---
layout: post
title: Checked Cast vs Convert in C#
---

Let's say we want to convert `int` to `short` with a bounds check. Is there a difference between `checked((short)x)` and `Convert.ToInt16(x)`?

# Behavior

The behavior is the same for both methods: if the value is out of bounds, an `OverflowException` is thrown.

# Performance

[Benchmark](https://gist.github.com/ptupitsyn/3e69d0938f02e8ec95dfe67d608a6561) results:

```
|      Method |        Val |          Mean |      Error |     StdDev | Ratio | RatioSD |
|------------ |----------- |--------------:|-----------:|-----------:|------:|--------:|
| CheckedCast |          1 |     0.3265 ns |  0.0030 ns |  0.0028 ns |  1.00 |    0.00 |
|   ConvertTo |          1 |     1.1499 ns |  0.0044 ns |  0.0042 ns |  3.52 |    0.03 |
|             |            |               |            |            |       |         |
| CheckedCast | 2147483647 | 6,430.8317 ns | 10.7705 ns | 10.0747 ns |  1.00 |    0.00 |
|   ConvertTo | 2147483647 | 6,405.4382 ns | 20.2827 ns | 18.9725 ns |  1.00 |    0.00 |
```

Checked cast is 3.5 times faster when there is no exception.

# Generated Code

Checked cast is compiled into a single [conv.ovf.i2](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.emit.opcodes.conv_ovf_i2?view=net-7.0) instruction.

`Convert.ToInt16` method uses an explicit bounds check and then an unchecked cast (`conv.i2`):
```csharp
public static short ToInt16(int value)
{
    if (value < short.MinValue || value > short.MaxValue) ThrowInt16OverflowException();
    return (short)value;
}
```

# Conclusion

With `Convert.ToInt16(x)` it is not immediately clear if the bounds check is performed or not. 

Checked cast is faster, generates less code, and carries the intent better.

