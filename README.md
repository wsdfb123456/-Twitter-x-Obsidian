# -Twitter-x-Obsidian

打开chrome浏览器并且登录Twitter/x后，可一键获取推文和下载相关图片到本地Obsidian里。
利用了one drive同步功能。


## 第一步：重新创建项目

建议新建一个干净的目录，避免旧文件干扰。

```cmd
C:\claude-x-saver
```

进入该目录：

```cmd
cd C:\claude-x-saver
```

## 第二步：安装依赖

在项目目录下打开命令提示符或 PowerShell，依次执行：

```cmd
npm init -y
```

```cmd
npm install playwright fs-extra slugify
```

> `playwright` 用于自动化浏览器操作，`fs-extra` 增强文件系统能力，`slugify` 用于生成安全的文件名。

## 第三步：安装浏览器

Playwright 需要单独安装 Chromium 浏览器：

```cmd
npx playwright install chromium
```

等待下载完成即可。

## 第四步：创建数据目录

创建一个用于保存 Playwright 独立浏览器数据的目录，这样登录状态会永久保存，不必每次重复登录。

```cmd
C:\claude-x-saver\data
```

你可以手动在资源管理器里创建，也可以在命令行执行：

```cmd
mkdir data
```

## 第五步：创建 `save-x.js` 文件

在 `C:\claude-x-saver` 目录下新建一个文件，命名为 `save-x.js`（注意大小写和扩展名）。

然后将下面的完整代码**完整复制**到该文件中：

```javascript
const { chromium } = require('playwright');
const fs = require('fs-extra');
const path = require('path');

// ======================
// 配置区域
// ======================

// Markdown保存目录，请设置在one drive同步文件夹里，先自己去创建对应的文件夹
const OBSIDIAN_PATH =
    "C:/Users/xxxxxxxxxxxxxxxxx改成你自己的用户名/OneDrive/Apps/remotely-save/Obsidian Vault";

// 图片保存目录
const IMAGE_PATH =
    "C:/Users/xxxxxxxxxxxxxxxxx改成你自己的用户名/OneDrive/Apps/remotely-save/Obsidian Vault/assets/twitter";

// Chrome配置目录
const USER_DATA_DIR =
    "C:/chrome-x-profile";

// ======================
// 时间文件名
// ======================

function getTimeFilename() {

    const now = new Date();

    const year = now.getFullYear();

    const month =
        String(now.getMonth() + 1)
            .padStart(2, '0');

    const day =
        String(now.getDate())
            .padStart(2, '0');

    const hour =
        String(now.getHours())
            .padStart(2, '0');

    const minute =
        String(now.getMinutes())
            .padStart(2, '0');

    const second =
        String(now.getSeconds())
            .padStart(2, '0');

    return `${year}-${month}-${day}-${hour}-${minute}-${second}`;
}

// ======================
// 转高清原图URL
// ======================

function getOriginalImageUrl(url) {

    // 删除旧name参数
    url = url.replace(
        /name=\w+/,
        'name=orig'
    );

    // 如果没有name参数
    if (!url.includes('name=')) {

        if (url.includes('?')) {
            url += '&name=orig';
        } else {
            url += '?name=orig';
        }
    }

    return url;
}

// ======================
// 下载图片
// ======================

async function downloadImage(
    page,
    url,
    savePath
) {

    const buffer = await page.evaluate(
        async (url) => {

            const response =
                await fetch(url);

            const blob =
                await response.blob();

            const arrayBuffer =
                await blob.arrayBuffer();

            return Array.from(
                new Uint8Array(arrayBuffer)
            );

        },
        url
    );

    await fs.writeFile(
        savePath,
        Buffer.from(buffer)
    );
}

// ======================
// 主程序
// ======================

async function main() {

    const url = process.argv[2];

    if (!url) {

        console.log("请提供Twitter/X链接");

        return;
    }

    console.log("启动Chrome...");

    const browser =
        await chromium.launchPersistentContext(
            USER_DATA_DIR,
            {
                headless: false,

                channel: 'chrome',

                args: [
                    '--disable-blink-features=AutomationControlled'
                ]
            }
        );

    const page =
        await browser.newPage();

    console.log("打开页面...");

    await page.goto(url, {
        waitUntil: 'domcontentloaded',
        timeout: 60000
    });

    console.log("等待页面加载...");

    await page.waitForTimeout(8000);

    // 登录检测
    if (
        page.url().includes('login')
    ) {

        console.log("");
        console.log("请先手动登录 X/Twitter");
        console.log("");

        await page.waitForTimeout(15000);
    }

    console.log("等待推文加载...");

    await page.waitForSelector(
        'article',
        {
            timeout: 30000
        }
    );

    console.log("提取推文...");

    const data =
        await page.evaluate(() => {

            const article =
                document.querySelector('article');

            if (!article) {
                return null;
            }

            const text =
                article.innerText;

            const images =
                Array.from(
                    article.querySelectorAll('img')
                )
                .map(img => img.src)
                .filter(src =>
                    src.includes(
                        'pbs.twimg.com/media'
                    )
                );

            return {
                text,
                images
            };
        });

    if (!data) {

        console.log("未找到推文");

        return;
    }

    // 创建目录
    await fs.ensureDir(
        OBSIDIAN_PATH
    );

    await fs.ensureDir(
        IMAGE_PATH
    );

    // 文件名
    const filename =
        getTimeFilename();

    // Markdown
    let md = '';

    md += `# Twitter收藏\n\n`;

    md += `原链接：${url}\n\n`;

    md += `保存时间：${new Date().toLocaleString()}\n\n`;

    md += `---\n\n`;

    md += `${data.text}\n\n`;

    // ======================
    // 下载高清图片
    // ======================

    if (data.images.length > 0) {

        md += `## 图片\n\n`;

        for (
            let i = 0;
            i < data.images.length;
            i++
        ) {

            let imgUrl =
                data.images[i];

            // 转高清原图
            imgUrl =
                getOriginalImageUrl(
                    imgUrl
                );

            console.log("");
            console.log(
                `高清图片URL:`
            );

            console.log(imgUrl);

            let ext = 'jpg';

            if (
                imgUrl.includes(
                    'format=png'
                )
            ) {
                ext = 'png';
            }

            if (
                imgUrl.includes(
                    'format=webp'
                )
            ) {
                ext = 'webp';
            }

            const imgName =
                `${filename}-${i + 1}.${ext}`;

            const imgPath =
                path.join(
                    IMAGE_PATH,
                    imgName
                );

            console.log(
                `下载高清图片 ${i + 1}`
            );

            try {

                await downloadImage(
                    page,
                    imgUrl,
                    imgPath
                );

                const stat =
                    await fs.stat(
                        imgPath
                    );

                console.log(
                    `图片大小: ${
                        Math.round(
                            stat.size / 1024
                        )
                    }KB`
                );

                md += `![[${imgName}]]\n\n`;

            } catch (err) {

                console.log(
                    "图片下载失败"
                );

                console.log(err);
            }
        }
    }

    // 保存Markdown
    const mdPath =
        path.join(
            OBSIDIAN_PATH,
            `${filename}.md`
        );

    await fs.writeFile(
        mdPath,
        md,
        'utf-8'
    );

    console.log("");
    console.log("=================================");
    console.log("保存完成");
    console.log(mdPath);
    console.log("=================================");
    console.log("");

}

main();

先打开chrome浏览器登录Twitter/x，然后关闭浏览器。
在C:\claude-x-saver目录下运行

```cmd
node save-x.js "https://x.com/............/"
