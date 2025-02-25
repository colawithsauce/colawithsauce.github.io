+++
title = "MLIR 如何写 Transform Dialect?"
author = ["colawithsauce"]
date = 2025-02-25T18:30:00+08:00
draft = false
+++

## 一些坑 {#一些坑}

如果要使用 transform-interpreter，那就最好不要先运行别的 Pass 再运行 transform-interpreter，因为大部分的 transform.\* Op 都会被认为是死代码而被 LLVM 消除。


## transform.foreach_matched {#transform-dot-foreach-matched}

```mlir
transform.foreach_matched in %a @match_xxx -> @handle_xxx, @match_yyy -> @handle_yyy
        : (!transform.any_op) -> !transform.any_op
```

首要注意的是，这个 Op 取 %a 是一个 candidate list，而其中的 `@match` 函数却不是匹配 %a 中的 candidate 本身，而是去 walk 匹配 candidate 中的内容。比如说下面这个例子

```mlir
linalg.genric {...} {
^bb0(...):
    %0 = arith.add %a, %b : f32
    linalg.yield %0 : f32
}
linalg.genric {...} {
^bb0(...):
    %0 = arith.mul %a, %b : f32
    linalg.yield %0 : f32
}
```

如果我们

```mlir
transform.named_sequence @__transform_main(%arg0: !transform.any_op {transform.consumed}) {
    %0 = transform.structure.match {["linalg.generic"]} in %arg0 : (!transform.any_op) -> (!transform.any_op)
    %tiled = transform.foreach_match in %0
      @match_xxx -> @tile_xxx
      : (!transform.any_op) -> !transform.any_op
    transform.yield
}
```

那么 `@match_xxx` 的输入不是上面最顶层的两个 `linalg.generic` 而是：

```mlir
arith.add ...
yield

arith.mul ...
yield
```

非常地反直觉。

附一个 MLIR 匹配一维与二维做不同的处理的 Transform 脚本

```mlir
module attributes {transform.with_named_sequence} {
  transform.named_sequence @match_1d_linalg_generic(%candidate: !transform.any_op {transform.readonly})
    -> !transform.any_op {
    %matched = transform.match.structured failures(propagate) %candidate : (!transform.any_op) -> !transform.any_op {
    ^bb0(%arg0: !transform.any_op):
      // with rank 1
      %rank = transform.match.structured.rank %arg0
      : (!transform.any_op) -> !transform.param<i64>
      %c1 = transform.param.constant 1 : i64 -> !transform.param<i64>
      transform.match.param.cmpi eq %rank, %c1 : !transform.param<i64>
      transform.match.structured.yield %candidate : !transform.any_op
    }
    transform.yield %matched : !transform.any_op
  }
  transform.named_sequence @match_2d_linalg_generic(%candidate: !transform.any_op {transform.readonly})
    -> !transform.any_op {
    %matched = transform.match.structured failures(propagate) %candidate : (!transform.any_op) -> !transform.any_op {
    ^bb0(%arg0: !transform.any_op):
      // with rank 2
      %rank = transform.match.structured.rank %arg0
      : (!transform.any_op) -> !transform.param<i64>
      %c2 = transform.param.constant 2 : i64 -> !transform.param<i64>
      transform.match.param.cmpi eq %rank, %c2 : !transform.param<i64>
      transform.match.structured.yield %candidate : !transform.any_op
    }
    transform.yield %matched: !transform.any_op
  }
  transform.named_sequence @tile_1d_linalg_generic(%candidate: !transform.any_op {transform.readonly}) {
    transform.debug.emit_remark_at %candidate ,"tiling 1d" : !transform.any_op
    %0 = transform.structured.match ops{["linalg.generic"]} in %candidate : (!transform.any_op) -> !transform.any_op
    %tiled_linalg_op, %loops = transform.structured.tile_using_for %0 tile_sizes [64] : (!transform.any_op) -> (!transform.any_op, !transform.any_op)
    transform.yield
  }
  transform.named_sequence @tile_2d_linalg_generic(%candidate: !transform.any_op {transform.readonly}) {
    transform.debug.emit_remark_at %candidate ,"tiling 2d" : !transform.any_op
    %0 = transform.structured.match ops{["linalg.generic"]} in %candidate : (!transform.any_op) -> !transform.any_op
    %tiled_linalg_op, %loops:2 = transform.structured.tile_using_for %0 tile_sizes [1, 64] : (!transform.any_op) -> (!transform.any_op, !transform.any_op, !transform.any_op)
    transform.yield
  }
  transform.named_sequence @__transform_main(%arg0: !transform.any_op {transform.consumed}) {
    %tiled = transform.foreach_match in %arg0
      @match_1d_linalg_generic -> @tile_1d_linalg_generic,
      @match_2d_linalg_generic -> @tile_2d_linalg_generic
      : (!transform.any_op) -> !transform.any_op
    //// vectorize
    //transform.structured.vectorize %tiled: !transform.any_op
    //// bufferization
    //transform.bufferization.eliminate_empty_tensors %arg0 : !transform.any_op
    //%1 = transform.bufferization.one_shot_bufferize
    //    layout{IdentityLayoutMap} %arg0
    //    {allow_return_allocs = true, bufferize_function_boundaries = true} :
    //  (!transform.any_op) -> !transform.any_op
    //transform.bufferization.buffer_loop_hoisting %1 : !transform.any_op

    //%2 = transform.structured.match ops{["func.func"]} in %1 :
    //  (!transform.any_op) -> !transform.any_op
    //transform.apply_patterns to %2 {
    //  // 把 1x64xf32 的 elemwise 运算转成 64xf32
    //  transform.apply_patterns.vector.drop_unit_dims_with_shape_cast
    //  transform.apply_patterns.vector.lower_shape_cast // 把 shape_cast 转成 extract/insert
    //  transform.apply_patterns.vector.rank_reducing_subview_patterns
    //  // transform.apply_patterns.vector.transfer_to_scf // 把 1x64xf32 的读转成 64xf32 的读
    //  transform.apply_patterns.vector.lower_transfer max_transfer_rank = 1
    //} : !transform.any_op
    transform.yield
  }
}
```
