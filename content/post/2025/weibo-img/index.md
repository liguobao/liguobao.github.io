---
title: 一行代码下载微博相册
description: weibo
slug: one-line-download-weibo-img
date: 2025-01-23 08:00:00+0000
image: logo.png
categories:
  - TC
tags:
  - 2025
  - weibo
  - 爬虫
---


```js
// ==UserScript==
// @name         下载微博相册图片为ZIP文件
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  下载微博相册上的所有图片并将它们打包为ZIP文件
// @author       You
// @match        https://weibo.com/u/*?tabtype=album
// @grant        GM_addStyle
// ==/UserScript==

(function() {
    'use strict';

    // 获取微博用户的 ID 作为下载文件名
    const match = window.location.href.match(/https:\/\/weibo\.com\/u\/(\d+)\?/);
    if (!match) {
        console.error("无法获取微博用户 ID");
        return;
    }
    const userId = match[1];
    const downloadFileName = `weibo_album_${userId}.zip`;

    // 插入按钮到页面顶部并居中
    const button = document.createElement("button");
    button.innerText = "下载图片为ZIP";
    button.style.position = "fixed";
    button.style.top = "50%";  // 垂直居中
    button.style.left = "50%";  // 水平居中
    button.style.transform = "translate(-50%, -50%)";  // 使按钮完全居中
    button.style.zIndex = "9999";  // 确保按钮位于页面顶部
    button.style.padding = "10px 20px";
    button.style.backgroundColor = "#007bff";
    button.style.color = "#fff";
    button.style.border = "none";
    button.style.borderRadius = "5px";
    button.style.cursor = "pointer";
    button.style.fontSize = "16px";  // 增加字体大小
    button.style.boxShadow = "0 2px 10px rgba(0, 0, 0, 0.3)";  // 添加阴影
    document.body.appendChild(button);

    // 按钮点击事件
    button.addEventListener("click", () => {
        downloadAllImagesAsZip();
    });

    // 加载 JSZip 库
    const loadJSZip = () => {
        return new Promise((resolve, reject) => {
            const script = document.createElement("script");
            script.src = "https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js";
            script.onload = resolve;
            script.onerror = reject;
            document.head.appendChild(script);
        });
    };

    // 下载所有图片并打包为ZIP
    const downloadAllImagesAsZip = async () => {
        console.log("开始加载 JSZip...");
        await loadJSZip(); // 加载 JSZip
        console.log("JSZip 加载完成。");

        const zip = new JSZip();  // 使用 new JSZip() 实例化
        const images = document.getElementsByClassName("woo-picture-img");

        console.log(`共找到 ${images.length} 张图片。`);

        if (images.length === 0) {
            alert("没有找到任何图片！");  // 如果没有图片，提示用户
            return;  // 终止函数，不执行下载逻辑
        }

        // 下载所有图片并存储到内存中
        const imageBlobs = await Promise.all(
            Array.from(images).map((img, index) => {
                if (img.tagName === "IMG" && img.src) {
                    console.log(`开始下载图片 ${index + 1}: ${img.src}`);
                    return fetch(img.src)
                        .then((response) => {
                            if (!response.ok) {
                                throw new Error(`HTTP 状态码: ${response.status}`);
                            }
                            return response.blob();
                        })
                        .then((blob) => {
                            console.log(`图片 ${index + 1} 下载完成。`);
                            return { blob, index, src: img.src };
                        })
                        .catch((error) => {
                            console.error(`无法下载图片 ${index + 1}: ${img.src}`, error);
                            return null; // 返回 null 表示下载失败
                        });
                } else {
                    console.warn(`图片元素无效: ${img}`);
                    return null;
                }
            })
        );

        console.log("所有图片下载完成，开始添加到 ZIP 包。");

        // 将图片添加到 ZIP 包
        imageBlobs.forEach((image) => {
            if (image) {
                const ext = image.src.split(".").pop().split("?")[0] || "jpg"; // 提取扩展名
                zip.file(`image-${image.index + 1}.${ext}`, image.blob); // 添加到 ZIP
                console.log(`图片 ${image.index + 1} 已添加到 ZIP 包。`);
            }
        });

        // 生成 ZIP 并下载
        console.log("开始生成 ZIP 文件。");
        zip.generateAsync({ type: "blob" }).then((content) => {
            console.log("ZIP 文件生成完成，开始下载。");
            const a = document.createElement("a");
            a.href = URL.createObjectURL(content);
            a.download = downloadFileName;  // 使用用户 ID 作为下载文件名
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            console.log("ZIP 文件下载完成。");
        });
    };
})();

```

效果嘛。

![zip-download.png](/img/post/one-line-download-weibo-img/zip-download.png)

大概会遇到CORS错误。

解决办法嘛 -> Modheader 修改一下ResponseHeader。

Access-Control-Allow-Origin 直接填成 *

完事。

下载内容。

![result-show.png](/img/post/one-line-download-weibo-img/result-show.png)
