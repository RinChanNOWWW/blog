---
title: 关于 Parquet Encodings 相关的一些记录
date: 2023-10-01 11:51:09
tags:
    - Parquet
    - 编码
categories:
    - 源码笔记
---

最近在看 Parquet 支持的各种编码的详细的实现，但是发现官方文档中对于一些细节的描述并不是很清楚，在此结合源码记录一下。

<!-- more -->

- 官方文档：https://github.com/apache/parquet-format/blob/master/Encodings.md
- 参考源码：https://github.com/apache/arrow-rs/tree/master/parquet/src/encodings


## PLAIN

### 如何编码变长字符串（BYTE_ARRAY）

每一个字符串都由 `<len><data>` 组成，其中 `<len>` 为定长的 `u32` 类型。直接上代码：

```rust
impl ParquetValueType for super::ByteArray {
    const PHYSICAL_TYPE: Type = Type::BYTE_ARRAY;

    #[inline]
    fn encode<W: std::io::Write>(
        values: &[Self],
        writer: &mut W,
        _: &mut BitWriter,
    ) -> Result<()> {
        for value in values {
            let len: u32 = value.len().try_into().unwrap();
            writer.write_all(&len.to_ne_bytes())?;
            let raw = value.data();
            writer.write_all(raw)?;
        }
        Ok(())
    }
}
```

P.S. 对于定长字符串（FIXED_LEN_BYTE_ARRAY），不需要把 `<len>` 一起编码，只需要编码 `<data>`。

默认情况下，字符串不会使用这样的编码，默认是采用 [DELTA_LENGTH_BYTE_ARRAY](#DELTA-LENGTH-BYTE-ARRAY) 方式。

## PLAIN_DICTIONARY / RLE_DICTIONARY 

没什么好补充的。

## RLE (RLE / Bit-Packing Hybrid)

### Bit width 如何选择

通过上层传入，Encoder 本身不会决定位宽。上层使用能包含当前所有数字的最小 bit width 作为 encoder 采用的 bit width。
用 LevelsEncoder 为例，在编解码过程中，我们都能够获取到 `max_level`，再通过这个值换算出最少需要的 bit 个数即可。

### 如何选择采用 RLE 还是 Bit-Packing 编码

编码时以 8 个 value 为一组，如果 8 个全部为同样的 value，则采用 RLE，并一直编码到不同的 value 出现为止；如果 8 个 value 中有不同的 value，则采用 Bit-Packing 编码。

即使 8 个 value 中只有第一个 value 不同，后面 7 个 value 都相同，也会采用 Bit-Packing 编码。如果下一个 value 和前面的 7 个 value 相同，但前面 8 个 value 已经作为一组数据进行了 Bit-Packing 编码，
这个后续的 value 也无法和前面的 value 一起被按照 RLE 编码了，只能作为新的一组数据重新选择编码方式。

举个例子，表示方式为：编码方式(值个数)

```
01111111    111111111111    00000001    10101010
BP(8)       RLE(12)         BP(8)       BP(8)
```

### 结束编码时 buffer 内不满 8 个 value 如何选择编码方式

如果 buffer 中的数据原不相同，则采用 RLE 编码；否则使用 0 padding 到 8 个之后使用 Bit-Packing 编码。

## BIT_PACKED

没什么好补充的。

## DELTA_BINARY_PACKED

如何确定 `block size`（一个 block 中包含的记录数）, `miniblock size`（一个 miniblock 包含的记录数）, `number of minibocks in a block`（一个 block 中 miniblock 的个数）：

直接上代码。

```rust
let mini_block_size = match T::T::PHYSICAL_TYPE {
    Type::INT32 => 32,
    Type::INT64 => 64,
    _ => unreachable!(),
};

let num_mini_blocks = DEFAULT_NUM_MINI_BLOCKS // 此值为硬编码常量 4。
let block_size = mini_block_size * num_mini_blocks;
assert_eq!(block_size % 128, 0);
```

## DELTA_LENGTH_BYTE_ARRAY

没什么好补充的。

## DELTA_BYTE_ARRAY

没什么好补充的。

## BYTE_STREAM_SPLIT

没什么好补充的。