---
title: One Line Code to Download Weibo Album
description: weibo
slug: one-line-download-weibo-img
date: 2025-01-23 08:00:00+0000
image: logo.png
categories:
  - TC
tags:
  - 2025
  - weibo
  - crawler
---

```js
// ==UserScript==
// @name         Download Weibo Album Images as ZIP File
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  Download all images from Weibo album and package them as ZIP file
// @author       You
// @match        https://weibo.com/u/*?tabtype=album
// @grant        GM_addStyle
// ==/UserScript==

(function() {
    'use strict';

    // Get Weibo user's ID as download filename
    const match = window.location.href.match(/https:\/\/weibo\.com\/u\/(\d+)\?/);
    if (!match) {
        console.error("Unable to get Weibo user ID");
        return;
    }
    const userId = match[1];
    const downloadFileName = `weibo_album_${userId}.zip`;

    // Insert button to page top and center it
    const button = document.createElement("button");
    button.innerText = "Download Images as ZIP";
    button.style.position = "fixed";
    button.style.top = "50%";  // Vertical center
    button.style.left = "50%";  // Horizontal center
    button.style.transform = "translate(-50%, -50%)";  // Make button completely centered
    button.style.zIndex = "9999";  // Ensure button is at page top
    button.style.padding = "10px 20px";
    button.style.backgroundColor = "#007bff";
    button.style.color = "#fff";
    button.style.border = "none";
    button.style.borderRadius = "5px";
    button.style.cursor = "pointer";
    button.style.fontSize = "16px";  // Increase font size
    button.style.boxShadow = "0 2px 10px rgba(0, 0, 0, 0.3)";  // Add shadow
    document.body.appendChild(button);

    // Button click event
    button.addEventListener("click", () => {
        downloadAllImagesAsZip();
    });

    // Load JSZip library
    const loadJSZip = () => {
        return new Promise((resolve, reject) => {
            const script = document.createElement("script");
            script.src = "https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js";
            script.onload = resolve;
            script.onerror = reject;
            document.head.appendChild(script);
        });
    };

    // Download all images and package as ZIP
    const downloadAllImagesAsZip = async () => {
        console.log("Starting to load JSZip...");
        await loadJSZip(); // Load JSZip
        console.log("JSZip loaded successfully.");

        const zip = new JSZip();  // Use new JSZip() to instantiate
        const images = document.getElementsByClassName("woo-picture-img");

        console.log(`Found ${images.length} images in total.`);

        if (images.length === 0) {
            alert("No images found!");  // If no images, alert user
            return;  // Terminate function, do not execute download logic
        }

        // Download all images and store in memory
        const imageBlobs = await Promise.all(
            Array.from(images).map((img, index) => {
                if (img.tagName === "IMG" && img.src) {
                    console.log(`Starting to download image ${index + 1}: ${img.src}`);
                    return fetch(img.src)
                        .then((response) => {
                            if (!response.ok) {
                                throw new Error(`HTTP status code: ${response.status}`);
                            }
                            return response.blob();
                        })
                        .then((blob) => {
                            console.log(`Image ${index + 1} download completed.`);
                            return { blob, index, src: img.src };
                        })
                        .catch((error) => {
                            console.error(`Unable to download image ${index + 1}: ${img.src}`, error);
                            return null; // Return null means download failed
                        });
                } else {
                    console.warn(`Invalid image element: ${img}`);
                    return null;
                }
            })
        );

        console.log("All images downloaded, starting to add to ZIP package.");

        // Add images to ZIP package
        imageBlobs.forEach((image) => {
            if (image) {
                const ext = image.src.split(".").pop().split("?")[0] || "jpg"; // Extract extension
                zip.file(`image-${image.index + 1}.${ext}`, image.blob); // Add to ZIP
                console.log(`Image ${image.index + 1} added to ZIP package.`);
            }
        });

        // Generate ZIP and download
        console.log("Starting to generate ZIP file.");
        zip.generateAsync({ type: "blob" }).then((content) => {
            console.log("ZIP file generation completed, starting download.");
            const a = document.createElement("a");
            a.href = URL.createObjectURL(content);
            a.download = downloadFileName;  // Use user ID as download filename
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            console.log("ZIP file download completed.");
        });
    };
})();

```

The effect.

![zip-download.png](/img/post/one-line-download-weibo-img/zip-download.png)

Might encounter CORS error.

Solution -> Use Modheader to modify ResponseHeader.

Set Access-Control-Allow-Origin to *

Done.

Download content.

![result-show.png](/img/post/one-line-download-weibo-img/result-show.png)
