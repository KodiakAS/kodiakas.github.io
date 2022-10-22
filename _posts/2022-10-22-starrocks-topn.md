---
layout: post
title: "Starrocks TOPN"
subtitle: "starrocks topn 排序算法解析"
author: "KodiakAS"
header-img: "img/database-bg.png"
header-mask: 0.3
hidden: true
tags:
  - 算法
  - 数据库
  - c++
---

## 相关代码位置
```
// Sort Chunks in memory with specified order by rules.
class ChunksSorterTopn : public ChunksSorter
```
chunks_sorter_topn.h
chunks_sorter_topn.cpp

Status ChunksSorterTopn::update(RuntimeState* state, const ChunkPtr& chunk)
Status ChunksSorterTopn::_sort_chunks(RuntimeState* state)

## 关键思路解析

### 1. 向buffer添加数据并启动排序
```c++
// 将chunk添加到排序buffer中，如果buffer中chunk数量超过limit（也就是topn，说明不需要那么多数据）
// 或者超过buffer大小（内存占用过多），就要进行一次排序，淘汰掉一部分数据
Status ChunksSorterTopn::update(RuntimeState* state, const ChunkPtr& chunk) {
    auto& raw_chunks = _raw_chunks.chunks;
    size_t chunk_number = raw_chunks.size();
    if (chunk_number <= 0) {
        raw_chunks.push_back(chunk);
        chunk_number++;
    } else if (raw_chunks[chunk_number - 1]->num_rows() + chunk->num_rows() > _state->chunk_size()) {
        raw_chunks.push_back(chunk);
        chunk_number++;
    } else {
        // Old planner will not remove duplicated sort column.
        // columns in chunk may have same column ptr
        // append_safe will check size of all columns in dest chunk
        // to ensure same column will not apppend repeatedly.
        raw_chunks[chunk_number - 1]->append_safe(*chunk);
    }
    _raw_chunks.size_of_rows += chunk->num_rows();

    // when number of Chunks exceeds _limit or _size_of_chunk_batch, run sort and then part of
    // cached chunks can be dropped, so it can reduce the memory usage.
    // TopN caches _limit or _size_of_chunk_batch primitive chunks,
    // performs sorting once, and discards extra rows

    if (_limit > 0 && (chunk_number >= _limit || chunk_number >= _max_buffered_chunks)) {
        // 进入排序逻辑
        RETURN_IF_ERROR(_sort_chunks(state));
    }

    return Status::OK();
}
```

### 2. 排序
```c++
Status ChunksSorterTopn::_sort_chunks(RuntimeState* state) {
    const size_t chunk_size = _raw_chunks.size_of_rows;

    // chunks for this batch.
    // 每次排序一批数据，分批是按照上一步介绍的规则分的
    DataSegments segments;

    // permutations for this batch:
    // if _init_merged_segment == false, means this is first batch:
    //     permutations.first is empty, and permutations.second is contains this batch.
    // else if _init_merged_segment == true, means this is not first batch, _merged_segment[low_value, high_value] is not empty:
    //     permutations.first < low_value and low_value <= permutations.second < high_value(in the case of asc).
    // 记录排序结果的结构，first和second对应BEFORE和IN，都是vector，里面的元素是PermutationItem，
    // 记录着一行数据在哪个chunk的哪个行号
    // 如果是第一轮排序，也就是还没有排好的数据，fisrt是空，second就是本次排序的结果
    // 如果不是第一轮，详见下文_filter_and_sort_data
    std::pair<Permutation, Permutation> permutations;

    // step 1: extract datas from _raw_chunks into segments,
    // and initialize permutations.second when _init_merged_segment == false.
    RETURN_IF_ERROR(_build_sorting_data(state, permutations.second, segments));

    // step 2: filter batch-chunks as permutations.first and permutations.second when _init_merged_segment == true.
    // sort part chunks in permutations.first and permutations.second, if _init_merged_segment == false means permutations.first is empty.
    // 1) 如果不是第一轮排序，而且已排序数据已经达到topn，就从已排好的数据中拿到最大和最小值，
    // 将待排序的数据分成三块
    // BEFORE(-inf, sorted_min), IN[sorted_min, sorted_max), [sorted_max, +inf)
    // 第三块直接扔掉
    // 2) 如果没达到topn，只能使用sorted_min来将数据分成两块
    // BEFORE(-inf, sorted_min)，IN[sorted_min, +inf)
    //
    // 然后分别对这两块进行排序，排序的结果分别存入 permutations的first和second
    RETURN_IF_ERROR(_filter_and_sort_data(state, permutations, segments, chunk_size));

    // step 3:
    // (1) take permutations.first as BEFORE.
    // (2) take permutations.second merge-sort with _merged_segment as IN.
    // the result is _merged_segment as [BEFORE, IN].
    // 因为BEFORE都是比先前排序后的最小值还小的值，所以它肯定在最前面，
    // 只需要把IN和先前排好的数据merge就行了
    // 最终把排序并merge后的数据物化写入_merged_segment
    RETURN_IF_ERROR(_merge_sort_data_as_merged_segment(state, permutations, segments));

    return Status::OK();
}
```

### 排序内部逻辑
- 每次处理一批chunk
sort_vertical_chunks

- 循环地把这些chunk中的同一列都拿出来，调用
sort_vertical_columns
一次处理所有chunk中的这一列

- 构造一个VerticalColumnSorter，使用访问者模式实现
把这一列进行排序，结果写在permutations中（每条数据排序结果的chunk下标及行下标，最后一步才物化）

- 如果这次排序需要的结果数量小于总数据量则使用std::partial_sort（只要一部分），否则使用 （全量排）
先全量排first，如果不够topn，再排second（rows_to_sort - first_size）
if (UNLIKELY(limit > 0 && first_iter < limit && limit <= last_iter))

 std::partial_sort
此种算法适用于在大量元素中找出前N个最大的，底层使用堆排序实现
```
partial_sort(beg,mid,end,comp)
beg - 起始位置
mid - 中间位置
end - 结束位置
comp - 自定义less-than比较运算符
```
1. 对原始容器内区间为[first, middle)的元素执行 make_heap() 操作构造一个大顶堆
2. 然后遍历剩余区间[middle, last)中的元素，剩余区间的每个元素均与大顶堆的堆顶元素进行比较（大顶堆的堆顶元素为最大元素，该元素为第一个元素，很容易获得），若堆顶元素较大，则交换堆顶元素和遍历得到的元素值（pop_heap ），并重新调整该大顶堆以维持该堆为大顶堆（adjust_heap）
3. 遍历结束后，[first, middle)区间内的元素便是排名在前的m个元素，再对该堆做一次堆排序 sort_heap() 便可得到最后的结果

![partial_sort](/img/in-post/partial_sort.png)




