---
layout: post
title: "Starrocks 向量化执行引擎执行流程简述"
subtitle: ""
author: "KodiakAS"
header-img: "img/database-bg.png"
header-mask: 0.3
tags:
  - 数据库
---

starrocks的vectorized执行引擎采用的是**火山模型 + batch**的模式，即自顶向下地调用算子树每个节点的get_next方法，每次获取一批处理好的数据

此种模式也被称为**向量化模型（Vectorized / Batch Model）**，相对于传统按行处理的火山模型，优势在于大幅减少函数调用次数、更便于使用SIMD指令优化以及对cache更友好，也是目前OLAP引擎使用较多的一种模型

下图描述了典型向量化模型的执行流程

![vec](/img/in-post/database/vec-model.png)


vectorized执行引擎根据FE（近似于Coordinator Node）生成的执行计划分片（Fragment，相当于qtree引擎的Slice），在BE（近似于GNode）生成实际用于执行的算子树，本文将简要描述算子树的构建和执行过程

# 算子树构建

根据FE通过Thrift发送过来的单机执行计划构造算子树
```c++
Status PlanFragmentExecutor::prepare(const TExecPlanFragmentParams& request) {
    ...
    RETURN_IF_ERROR(ExecNode::create_tree(_runtime_state, obj_pool(), request.fragment.plan, *desc_tbl, &_plan));
    ...
}
```

最终会调用`ExecNode::create_vectorized_node`来根据参数实际构造算子树的节点，每个节点都是ExecNode（执行节点）的子类，在starrocks pipeline引擎及qtree引擎中被称为 operator（算子），以下统称为算子

```c++
Status ExecNode::create_vectorized_node(starrocks::RuntimeState* state, starrocks::ObjectPool* pool,
                                        const starrocks::TPlanNode& tnode, const starrocks::DescriptorTbl& descs,
                                        starrocks::ExecNode** node) {
    switch (tnode.node_type) {
    case TPlanNodeType::OLAP_SCAN_NODE:
        *node = pool->add(new vectorized::OlapScanNode(pool, tnode, descs));
        return Status::OK();
    case TPlanNodeType::META_SCAN_NODE:
        *node = pool->add(new vectorized::OlapMetaScanNode(pool, tnode, descs));
        return Status::OK();
    case TPlanNodeType::AGGREGATION_NODE:
    ...
    default:
        return Status::InternalError(strings::Substitute("Vectorized engine not support node: $0", tnode.node_type));
    }
}
```

# 算子树执行

算子树正式开始执行起始于`PlanFragmentExecutor::_open_internal_vectorized`，它会开启一个循环，不断地从算子树的根节点获取chunk，直到返回的chunk是一个空指针
```c++
Status PlanFragmentExecutor::_open_internal_vectorized() {

    // 首先会调用根节点的open方法来启动这个算子，每种算子的启动行为各不相同
    {
        SCOPED_TIMER(profile()->total_time_counter());
        RETURN_IF_ERROR(_plan->open(_runtime_state));
    }

    vectorized::ChunkPtr chunk;
    while (true) {
        RETURN_IF_ERROR(runtime_state()->check_mem_limit("QUERY"));

        // 开始执行
        RETURN_IF_ERROR(_get_next_internal_vectorized(&chunk));  // STEP INTO

        if (chunk == nullptr) {
            break;
        }

        // 各个 Fragment 的数据流转和最终的结果发送依赖 DataSink，比如 DataStreamSink 会将一个 Fragment 的数据发送到
        // 另一个Fragment 的 ExchangeNode，ResultSink 会将查询的结果集发送到 FE
        RETURN_IF_ERROR(_sink->send_chunk(runtime_state(), chunk.get()));
    }
    ...
}
```

上述函数会循环调用`PlanFragmentExecutor::_get_next_internal_vectorized`来驱动整个算子树的执行，自顶向下调用每个 ExecNode 的 get_next 方法，最终数据会从scan算子产生，封装成chunk向上一层层传递，每个算子都会按照自己的逻辑处理chunk

```c++
Status PlanFragmentExecutor::_get_next_internal_vectorized(vectorized::ChunkPtr* chunk) {

    // 如果读取到空的chunk，则进入下一轮循环，继续读取chunk
    while (!_done) {
        SCOPED_TIMER(profile()->total_time_counter());

        // 调用算子树根节点的get_next方法，将获取到的数据写入_chunk，将下层算子是否还有数据需要处理（bool）写入_done
        RETURN_IF_ERROR(_plan->get_next(_runtime_state, &_chunk, &_done));  // STEP INTO

        if (_done) {
            // 如果下层算子数据全部处理完了，则将chunk置为空指针，告诉调用者这个Fragment已经执行完了
            *chunk = nullptr;
            return Status::OK();
        } else if (_chunk->num_rows() > 0) {
            // 如果没全部处理完，且读到的chunk中有数据，则更新计数器并返回
            COUNTER_UPDATE(_rows_produced_counter, _chunk->num_rows());
            *chunk = _chunk;
            return Status::OK();
        }
    }

    return Status::OK();
}
```

假设当前算子树的根节点是一个阻塞聚合算子，即在open阶段就循环调用children的get_next，获取到所有数据并构建哈希表。当自身的get_next被调用时，聚合算子已经拥有所有数据，仅根据需要对其进行输出
*注：starrocks同时还有流式聚合算子，用于预聚合操作，open的时候不从下层算子获取数据，而是每次get_next时获取并处理一批数据*

由于在之前的流程中已经被open，这里直接关注它的get_next方法

```c++
Status AggregateBlockingNode::get_next(RuntimeState* state, ChunkPtr* chunk, bool* eos) {

    // 如果哈希表中数据处理完了或者处理的数据行数已经达到limit，则将算子标记为完成并返回
    if (_aggregator->is_ht_eos()) {
        COUNTER_SET(_aggregator->rows_returned_counter(), _aggregator->num_rows_returned());
        *eos = true;
        return Status::OK();
    }
    int32_t chunk_size = runtime_state()->chunk_size();

    // 将数据写入空的chunk
    if (_aggregator->is_none_group_by_exprs()) {
        // 如果不需要group by，则调用convert_to_chunk_no_groupby将数据聚合并装入chunk中
        SCOPED_TIMER(_aggregator->get_results_timer());
        _aggregator->convert_to_chunk_no_groupby(chunk);
    } else {
        // 如果需要group by，则根据不同数据类型将哈希表中的数据聚合并装入chunk
        if (false) {
        }
#define HASH_MAP_METHOD(NAME)                                                                                     \
    else if (_aggregator->hash_map_variant().type == AggHashMapVariant::Type::NAME)                               \
            _aggregator->convert_hash_map_to_chunk<decltype(_aggregator->hash_map_variant().NAME)::element_type>( \
                    *_aggregator->hash_map_variant().NAME, chunk_size, chunk);
        APPLY_FOR_AGG_VARIANT_ALL(HASH_MAP_METHOD)
#undef HASH_MAP_METHOD
    }

    size_t old_size = (*chunk)->num_rows();
    eval_join_runtime_filters(chunk->get());

    // For having
    ExecNode::eval_conjuncts(_conjunct_ctxs, (*chunk).get());
    _aggregator->update_num_rows_returned(-(old_size - (*chunk)->num_rows()));

    // 检查聚合算子从创建开始，返回的总数据行数是否已经达到了limit，如果达到了则将_is_ht_eos设为true，
    // 下次调用get_next时会将算子标记为完成并返回
    _aggregator->process_limit(chunk);

    DCHECK_CHUNK(*chunk);

    RETURN_IF_ERROR(_aggregator->check_has_error());

    return Status::OK();
}
```

以上是一个聚合算子执行的主要流程，假设它的children是一个scan算子（实际上在聚合算子的open阶段就完成了任务），继续向下观察执行过程

scan算子是最底层的算子类型，负责提供原始数据。此处以OlapScanNode为例，即从OLAP存储引擎（doris和starrocks的存储引擎名字就叫olap，使用类似架构的impala此类节点名为KuduScanNode）获取数据

此种算子的执行过程较为特殊，涉及任务调度和线程池机制，此部分机制将另行描述

```c++
Status OlapScanNode::get_next(RuntimeState* state, ChunkPtr* chunk, bool* eos) {

    // 如果是第一次调用，就调用_start_scan方法来初始化scan任务，涉及scanner线程的启动，chunk_pool的初始化等
    bool first_call = !_start;
    if (!_start && _status.ok()) {
        Status status = _start_scan(state);
    }

    // 如果scan算子已经启动，且状态不为ok，则认为扫描完成了，返回
    Status status = _get_status();
    if (!status.ok()) {
        *eos = true;
        return status.is_end_of_file() ? Status::OK() : status;
    }

    // 根据当前运行的scanner线程数量及并行度上限，尝试让更多的scanner参与scan
    {
        if ((num_pending > 0) && (num_running < kMaxConcurrency)) {
            // 提交一个新的scanner之前，先检查_chunk_pool里的chunk是否够用
            if (_chunk_pool.size() >= (num_running + 1) * _chunks_per_scanner) {
                TabletScanner* scanner = _pending_scanners.pop();
                l.unlock();
                (void)_submit_scanner(scanner, true);
            }
        }
    }

    // _result_chunks是一个UnboundedBlockingQueue，用于存放scanner线程读取上来的chunk
    // 阻塞地从_result_chunks队列中获取一个chunk
    if (_result_chunks.blocking_get(chunk)) {
        TRY_CATCH_BAD_ALLOC(_fill_chunk_pool(1, first_call));
        eval_join_runtime_filters(chunk);
        _num_rows_returned += (*chunk)->num_rows();
        COUNTER_SET(_rows_returned_counter, _num_rows_returned);

        // 如果scan的总行数已经达到limit，则关闭队列并更新状态，下次get_next就会结束scan算子
        if (reached_limit()) {
            int64_t num_rows_over = _num_rows_returned - _limit;
            DCHECK_GE((*chunk)->num_rows(), num_rows_over);
            (*chunk)->set_num_rows((*chunk)->num_rows() - num_rows_over);
            COUNTER_SET(_rows_returned_counter, _limit);
            _update_status(Status::EndOfFile("OlapScanNode has reach limit"));
            _result_chunks.shutdown();
        }

        // 运行到这里说明scan算子的任务还没完成，返回状态
        *eos = false;
        DCHECK_CHUNK(*chunk);
        return Status::OK();
    }

    // 运行到这里说明scan算子的任务结束了，返回状态
    _update_status(Status::EndOfFile("EOF of OlapScanNode"));
    *eos = true;
    status = _get_status();
    return status.is_end_of_file() ? Status::OK() : status;
}
```
