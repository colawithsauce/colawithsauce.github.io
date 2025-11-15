+++
title = "如何在 MLIR 中优雅地创建 transform.named_sequence"
author = ["温曾年"]
date = 2025-11-14T22:00:00+08:00
tags = ["MLIR", [",", "Transform"], "Dialect"]
categories = ["Programming"]
draft = false
mathjax = false
+++

## 背景 {#背景}

目前正在做一个任务：使用 transform dialect 对 IR 进行调度，生成高性能算子。我们会需要先根据 IR 信息作规划，再生成 transform dialect 来生成调度。

目前的想法是，生成一个 \`transform.named_sequence\` 再在里面添加调度的 op 进行调度。

目前在生成 op 这一步卡住了，因为我发现我生成的 op 在打印的时候是不能 pretty print 的。我发现我少了一项重要的内容，就是没有打上 `transform.consumed` 或者 `transform.readonly` 的参数属性。

考虑到读者可能已经熟练掌握了 `rewriter.create` ，因此本教程不再教 `transform` 内部怎么写。因此下面只在技术上记录一下如何优雅地创建一个 `transform.named_sequence`


### transform.named_sequence 的定义 {#transform-dot-named-sequence-的定义}

在创建这个 op 前，需要知道它的参数定义。我们可以在 MLIR 官方文档中找到 `transform.named_sequence` 的定义：

| Attribute      | MLIR Type          | Description                    |
|----------------|--------------------|--------------------------------|
| sym_name       | ::mlir::StringAttr | string attribute               |
| function_type  | ::mlir::TypeAttr   | function type attribute        |
| sym_visibility | ::mlir::StringAttr | string attribute               |
| arg_attrs      | ::mlir::ArrayAttr  | Array of dictionary attributes |
| res_attrs      | ::mlir::ArrayAttr  | Array of dictionary attributes |

其中 sym_name 和 function_type 是必须的，其它属性可以不设置。这些属性各自的含义是：

-   `sym_name`: 这个 named_sequence 的名字
-   `function_type`: 这个 named_sequence 的函数签名
-   `sym_visibility`: 这个 named_sequence 的可见性，可以是 "public" 或 "private"
-   `arg_attrs`: 这个 named_sequence 的参数属性
-   `res_attrs`: 这个 named_sequence 的返回值属性


### 在 C++ 中创建 transform.named_sequence {#在-c-plus-plus-中创建-transform-dot-named-sequence}

在 `Transform.cpp` 文件里面，有一个创建 `transform.named_sequence` 的例子：

```cpp
void transform::NamedSequenceOp::build(OpBuilder &builder,
                                       OperationState &state, StringRef symName,
                                       Type rootType, TypeRange resultTypes,
                                       SequenceBodyBuilderFn bodyBuilder,
                                       ArrayRef<NamedAttribute> attrs,
                                       ArrayRef<DictionaryAttr> argAttrs) {
  state.addAttribute(SymbolTable::getSymbolAttrName(),
                     builder.getStringAttr(symName));
  state.addAttribute(getFunctionTypeAttrName(state.name),
                     TypeAttr::get(FunctionType::get(builder.getContext(),
                                                     rootType, resultTypes)));
  state.attributes.append(attrs.begin(), attrs.end());
  state.addRegion();

  buildSequenceBody(builder, state, rootType,
                    /*extraBindingTypes=*/TypeRange(), bodyBuilder);
}
```

似乎和前面提到的属性列表对应不上。其实是因为这是提供给我们使用的简化方法，把 rootType 和 resultTypes 暴露给我们，而不是让我们自己去构造 function_type 属性。

另外一个观察是，好像这个方法没有提供 arg_attrs 和 res_attrs 的参数。虽然有 argAttrs，但函数体里面并没有使用到它。此外，res_attrs 也没有出现在参数列表中。


### 我们期望创建的 transform.named_sequence {#我们期望创建的-transform-dot-named-sequence}

```mlir
  transform.named_sequence @__transform_main(%arg0: !transform.any_op {transform.consumed}) {
    // body
    transform.yield
  }
```

它中间包含的要素：

1.  名字是 `@__transform_main`
2.  参数是一个 `!transform.any_op` 类型，并且带有 `{transform.consumed}` 属性
3.  返回值是空的
4.  中间的 body 可以是任意的 transform dialect op

其中，参数的属性 `{transform.consumed}` 是必须的，否则打印出来的 op 不能 pretty print。聪明的读者可能已经把其它的属性搞清楚要怎么写了，就是：

```cpp
  transform::NamedSequenceOp seqOp =
      builder.create<transform::NamedSequenceOp>(
          funcOp.getLoc(),
          /*sym_name=*/kSequenceName, // string type
          /*rootType=*/builder.getType<transform::AnyOpType>(),
          /*resultType=*/TypeRange{},
          /*bodyBuilder=*/
          [](OpBuilder &b, Location nested, Value rootH) {
            b.create<transform::YieldOp>(nested, ValueRange());
          });
```

其实这样已经可以创建一个 `transform.named_sequence` 了，但是缺少了参数属性。不能够 pretty print。这里我也不卖关子了，直接告诉大家正确的写法：

```cpp
  transform::NamedSequenceOp seqOp =
      builder.create<transform::NamedSequenceOp>(
          funcOp.getLoc(),
          /*sym_name=*/kSequenceName, // string type
          /*rootType=*/builder.getType<transform::AnyOpType>(),
          /*resultType=*/TypeRange{},
          /*bodyBuilder=*/
          [](OpBuilder &b, Location nested, Value rootH) {
            b.create<transform::YieldOp>(nested, ValueRange());
          },
          /*args=*/ArrayRef<NamedAttribute>{},
          /*attrArgs*/
          ArrayRef<DictionaryAttr>{builder.getDictionaryAttr(
              ArrayRef<NamedAttribute>{builder.getNamedAttr(
                  transform::TransformDialect::kArgReadOnlyAttrName,
                  builder.getUnitAttr())})});
```

注意这里最后一个参数，它需要一个 DictionaryAttr 的数组，但是又其实不需要其 value 是什么内容，只需要有 key 就行了。我摸索了好久才发现可以用 unitAttr() 来创建一个空的 value。
