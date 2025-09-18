---
layout: post
title: Byte Array to String — Safer URL Transport in C#
category: dotnet
date: 2016-10-04
tags:
- dotnet core
---

Sometimes we encrypt data and send it over the network. A common flow is AES-256 → byte[] → string → transmit → decrypt on the other side.

In C#, we often convert a byte array to string with `Convert.ToBase64String()`:

```csharp
public string BytesToString(byte[] buff)
{
   return Convert.ToBase64String(buff);
}

public byte[] StringToBytes(string input)
{
    return Encoding.UTF8.GetBytes(input);
}
```

This usually works, but if you pass the data via URL, special characters like `+`, space, `%` can break query strings and cause mismatches, leading to decryption failures.

References (Chinese):

1) URL encoding basics: http://www.ruanyifeng.com/blog/2010/02/url_encoding.html
2) Handling `+`, space, `=`, `%`, `&`, `#` in URL params: http://blog.csdn.net/luo_deng/article/details/12186535

You might consider `HttpUtility.UrlEncode` / `HttpUtility.UrlDecode`. But beware: browsers may auto-decode once, and if your backend also decodes, you’ll end up decoding twice — corrupting the data.

Conclusion: `Convert.ToBase64String()` can yield URL-unsafe characters. One workaround: encode the bytes as hex strings (0–255 → two hex chars), avoiding reserved characters entirely.

```csharp
/// byte[] → hex string
private static string BytesToString(byte[] bytes)
{
    if (bytes == null) return string.Empty;
    return string.Join(string.Empty, bytes.Select(b => string.Format("{0:x2}", b)).ToArray());
}

/// hex string → byte[]
private static byte[] StringToBytes(string str)
{
    if (string.IsNullOrEmpty(str)) return null;
    byte[] bytes = new byte[str.Length / 2];
    for (int i = 0; i < str.Length; i += 2)
    {
        bytes[i / 2] = Convert.ToByte("0x" + str[i] + str[i + 1], 16);
    }
    return bytes;
}
```

Problem solved.

