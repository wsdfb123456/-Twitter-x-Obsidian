# 批量获取 Twitter/X 推文图片到 Obsidian，并同步至 OneDrive

本方案可以实现：**只要有 Twitter/X 链接**，就能批量抓取图文内容，自动保存到 Obsidian 笔记中，并利用 OneDrive 在多端同步。

## 准备工作 onedrive同步设置

### 1. 设置 OneDrive 与 Obsidian 目录（电脑端）

在 OneDrive 中创建或指定一个用于存放 Obsidian 库的文件夹，例如：

```plaintext
C:\Users\你的用户名\OneDrive\Apps\remotely-save\Obsidian Vault
```

### 2. 安装并配置 Obsidian 插件 `Remotely Save`（电脑端）

- 打开 Obsidian，搜索并安装插件 **Remotely Save**  
- 进入插件设置：
  - **服务类型** → 选择 `OneDrive 个人版`
  - 点击 **登录** 并授权你的 OneDrive 账号
  - **同步文件夹** → 填写刚刚设置的文件夹路径（如 `Obsidian Vault`）

### 3. 手机端配置（安卓 / iOS）

- 下载安装 Obsidian 移动端
- 同样安装 **Remotely Save** 插件
- 授权登录同一个 OneDrive 账号
- 创建**同名**的 Obsidian 库文件夹（例如 `Obsidian Vault`）

> ✅ 此时电脑与手机可以通过 OneDrive 自动同步该库中的内容。



## 自动获取推文图文（电脑端）

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

## 第五步：创建 `推文链接.txt` 文件

把Twitter/x链接填到里面，每行一个链接。
（如果使用Claude，可以创建txt以后跳过填写步骤）


## 第六步：创建自动运行的代码文件

# 1.如果想要批量保存多条链接

在 `C:\claude-x-saver` 目录下新建一个文件，命名为 `save-x.js`（注意大小写和扩展名）。

然后将下面的完整代码**完整复制**到该文件中：（记得更改“你的用户名”）

```
const { chromium } = require('playwright');
const fs = require('fs-extra');
const path = require('path');

// ======================
// 配置区域
// ======================

// Markdown保存目录
const OBSIDIAN_PATH =
    "C:/Users/你的用户名/OneDrive/Apps/remotely-save/Obsidian Vault";

// 图片保存目录
const IMAGE_PATH =
    "C:/Users/你的用户名/OneDrive/Apps/remotely-save/Obsidian Vault/assets/twitter";

// Chrome独立配置目录
const USER_DATA_DIR =
    "C:/chrome-x-profile";

// 批量链接文件
const LINKS_FILE =
    "推文链接.txt";

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

    url = url.replace(
        /name=\w+/,
        'name=orig'
    );

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
// 下载真实图片
// ======================

async function downloadImage(
    page,
    url,
    savePath
) {

    const buffer =
        await page.evaluate(
            async (url) => {

                const response =
                    await fetch(url);

                const blob =
                    await response.blob();

                const arrayBuffer =
                    await blob.arrayBuffer();

                return Array.from(
                    new Uint8Array(
                        arrayBuffer
                    )
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
// 处理单个推文
// ======================

async function processTweet(
    page,
    url
) {

    console.log("");
    console.log("=====================");
    console.log("处理推文:");
    console.log(url);
    console.log("=====================");
    console.log("");

    try {

        await page.goto(url, {
            waitUntil: 'domcontentloaded',
            timeout: 60000
        });

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

        await page.waitForSelector(
            'article',
            {
                timeout: 30000
            }
        );

        console.log("提取推文内容...");

        const data =
            await page.evaluate(() => {

                const article =
                    document.querySelector(
                        'article'
                    );

                if (!article) {
                    return null;
                }

                // 正文
                const text =
                    article.innerText;

                // 图片 + alt
                const images =
                    Array.from(
                        article.querySelectorAll(
                            'img'
                        )
                    )
                    .filter(img =>
                        img.src.includes(
                            'pbs.twimg.com/media'
                        )
                    )
                    .map(img => {

                        return {

                            url: img.src,

                            alt: img.alt || ''

                        };

                    });

                return {
                    text,
                    images
                };
            });

        if (!data) {

            console.log(
                "未找到推文"
            );

            return;
        }

        const filename =
            getTimeFilename();

        let md = '';

        md += `# Twitter收藏\n\n`;

        md += `原链接：${url}\n\n`;

        md += `保存时间：${new Date().toLocaleString()}\n\n`;

        md += `---\n\n`;

        md += `${data.text}\n\n`;

        // ======================
        // 下载图片
        // ======================

        if (
            data.images.length > 0
        ) {

            md += `## 图片\n\n`;

            for (
                let i = 0;
                i < data.images.length;
                i++
            ) {

                const imageData =
                    data.images[i];

                let imgUrl =
                    imageData.url;

                const alt =
                    imageData.alt;

                // 转原图
                imgUrl =
                    getOriginalImageUrl(
                        imgUrl
                    );

                console.log("");
                console.log(
                    `下载图片 ${i + 1}`
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
                        `图片大小 ${
                            Math.round(
                                stat.size / 1024
                            )
                        }KB`
                    );

                    // 写入Prompt
                    if (alt) {

                        md += `### 图片 ${i + 1} Prompt\n\n`;

                        md += "```text\n";

                        md += `${alt}\n`;

                        md += "```\n\n";
                    }

                    // 本地图片
                    md += `![[${imgName}]]\n\n`;

                } catch (err) {

                    console.log(
                        "图片下载失败"
                    );

                    console.log(err);
                }
            }
        }

        // ======================
        // 保存Markdown
        // ======================

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
        console.log("=====================");
        console.log("保存成功");
        console.log(mdPath);
        console.log("=====================");
        console.log("");

    } catch (err) {

        console.log("");
        console.log("=====================");
        console.log("处理失败");
        console.log(url);
        console.log("=====================");
        console.log("");

        console.log(err);
    }
}

// ======================
// 主程序
// ======================

async function main() {

    await fs.ensureDir(
        OBSIDIAN_PATH
    );

    await fs.ensureDir(
        IMAGE_PATH
    );

    // 读取链接
    const text =
        await fs.readFile(
            LINKS_FILE,
            'utf-8'
        );

    const links =
        text
            .split('\n')
            .map(v => v.trim())
            .filter(v => v);

    console.log("");
    console.log(
        `读取到 ${links.length} 个链接`
    );
    console.log("");

    // 启动浏览器
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

    // 批量处理
    for (const url of links) {

        await processTweet(
            page,
            url
        );

        // 防止风控
        await page.waitForTimeout(
            3000
        );
    }

    console.log("");
    console.log("=====================");
    console.log("全部处理完成");
    console.log("=====================");
    console.log("");

    await browser.close();
}

main();
```

# 12.如果想要只保存单条链接

在 `C:\claude-x-saver` 目录下新建一个文件，命名为 `save-x-one.js`（注意大小写和扩展名）。

然后将下面的完整代码**完整复制**到该文件中：（记得更改“你的用户名”）

```
const { chromium } = require('playwright');
const fs = require('fs-extra');
const path = require('path');

// ======================
// 配置区域
// ======================

// Markdown保存目录
const OBSIDIAN_PATH =
    "C:/Users/你的用户名/OneDrive/Apps/remotely-save/Obsidian Vault";

// 图片保存目录
const IMAGE_PATH =
    "C:/Users/你的用户名/OneDrive/Apps/remotely-save/Obsidian Vault/assets/twitter";

// Chrome独立配置目录
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
// 高清原图
// ======================

function getOriginalImageUrl(url) {

    url = url.replace(
        /name=\w+/,
        'name=orig'
    );

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
// 下载真实图片
// ======================

async function downloadImage(
    page,
    url,
    savePath
) {

    const buffer =
        await page.evaluate(
            async (url) => {

                const response =
                    await fetch(url);

                const blob =
                    await response.blob();

                const arrayBuffer =
                    await blob.arrayBuffer();

                return Array.from(
                    new Uint8Array(
                        arrayBuffer
                    )
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

    // 命令行链接
    const url =
        process.argv[2];

    if (!url) {

        console.log("");
        console.log("请提供推文链接");
        console.log("");
        console.log(
            '示例:'
        );
        console.log(
            'node save-x.js "https://x.com/xxx/status/123"'
        );
        console.log("");

        return;
    }

    // 创建目录
    await fs.ensureDir(
        OBSIDIAN_PATH
    );

    await fs.ensureDir(
        IMAGE_PATH
    );

    console.log("");
    console.log("=====================");
    console.log("启动Chrome");
    console.log("=====================");
    console.log("");

    // 启动浏览器
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

    try {

        console.log("");
        console.log("=====================");
        console.log("打开推文");
        console.log("=====================");
        console.log(url);
        console.log("");

        await page.goto(url, {
            waitUntil: 'domcontentloaded',
            timeout: 60000
        });

        await page.waitForTimeout(8000);

        // 登录检测
        if (
            page.url().includes('login')
        ) {

            console.log("");
            console.log(
                "请先手动登录 X/Twitter"
            );
            console.log("");

            await page.waitForTimeout(
                15000
            );
        }

        console.log("");
        console.log("=====================");
        console.log("等待推文加载");
        console.log("=====================");
        console.log("");

        await page.waitForSelector(
            'article',
            {
                timeout: 30000
            }
        );

        console.log("");
        console.log("=====================");
        console.log("提取推文内容");
        console.log("=====================");
        console.log("");

        const data =
            await page.evaluate(() => {

                const article =
                    document.querySelector(
                        'article'
                    );

                if (!article) {
                    return null;
                }

                // 正文
                const text =
                    article.innerText;

                // 图片 + alt
                const images =
                    Array.from(
                        article.querySelectorAll(
                            'img'
                        )
                    )
                    .filter(img =>
                        img.src.includes(
                            'pbs.twimg.com/media'
                        )
                    )
                    .map(img => {

                        return {

                            url: img.src,

                            alt: img.alt || ''

                        };

                    });

                return {
                    text,
                    images
                };
            });

        if (!data) {

            console.log(
                "未找到推文"
            );

            return;
        }

        // 文件名
        const filename =
            getTimeFilename();

        // Markdown内容
        let md = '';

        md += `# Twitter收藏\n\n`;

        md += `原链接：${url}\n\n`;

        md += `保存时间：${new Date().toLocaleString()}\n\n`;

        md += `---\n\n`;

        md += `${data.text}\n\n`;

        // ======================
        // 下载图片
        // ======================

        if (
            data.images.length > 0
        ) {

            md += `## 图片\n\n`;

            for (
                let i = 0;
                i < data.images.length;
                i++
            ) {

                const imageData =
                    data.images[i];

                let imgUrl =
                    imageData.url;

                const alt =
                    imageData.alt;

                // 高清原图
                imgUrl =
                    getOriginalImageUrl(
                        imgUrl
                    );

                console.log("");
                console.log(
                    `下载图片 ${i + 1}`
                );
                console.log("");

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
                        `图片大小 ${
                            Math.round(
                                stat.size / 1024
                            )
                        }KB`
                    );

                    // 保存Prompt
                    if (alt) {

                        md += `### 图片 ${i + 1} Prompt\n\n`;

                        md += "```text\n";

                        md += `${alt}\n`;

                        md += "```\n\n";
                    }

                    // Obsidian本地图片
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
        console.log("=====================");
        console.log("保存成功");
        console.log("=====================");
        console.log("");

        console.log(mdPath);

    } catch (err) {

        console.log("");
        console.log("=====================");
        console.log("运行失败");
        console.log("=====================");
        console.log("");

        console.log(err);

    } finally {

        await browser.close();
    }
}

main();
```

在当前目录运行

```cmd
node save-x-one.js "你的Twitter/x链接"
```


## Claude code skill




在Claude code skill 文件夹下创建以下skill.md

```
---
name: save-twitter
description: 保存Twitter/X推文到Obsidian，包括正文、高清图片、Prompt
---

//请改成你的实际存放地址
以下内容使用到的 推文链接.txt 为 x:/xxxxx/xxxxx/xxxxx/推文链接.txt
save-x.js 为 x:/xxxxx/xxxxx/xxxxx/save-x.js
save-x-one.js 为 x:/xxxxx/xxxxx/xxxxx/save-x-one.js


当用户要求：

- 保存推文
- 保存Twitter
- 保存X链接
- 批量保存推文
- 保存 推文链接.txt
且只提供单个Twitter/x链接

时：

执行：

node save-x-one.js "(用户提供的链接)"


如果用户提供多个Twitter/x链接：

1. 清空 推文链接.txt 里面的内容
将用户提供Twitter/x链接写入 推文链接.txt，每个链接一行
2. 执行：

node save-x.js

执行完成后：

告诉用户：

- 已保存到 Obsidian
- 图片已下载
- Prompt 已提取

接下来继续执行以下内容：

接下来的内容请你在这个Obsidian文件夹“C:\Users\你的用户名\OneDrive\Apps\remotely-save\Obsidian Vault”下完成工作，不要去其他文件夹。

请你帮我整理Obsidian笔记的内容。先判断文件名，请你只整理不带有中文文件名的.md文件（例如‘12-31-123-5550’），不要去改动文件名含有中文的.md文件（例如‘ai提示词123关注’）。
要求一：把笔记里面的多余的无关紧要的文字删除，保留需要的内容，（日期、链接、网名等不能删除），只保留有实质信息的内容，删除所有冗余、重复、无意义或社交媒体的元数据。
【保留规则】
1. 保留有明确含义的中文或英文句子（如作者观点、方法说明、链接、标签、用户名、ID、提示词内容）。
2. 保留格式类似“xxx: xxx”的键值对、列表项（如“GPT2: 自媒体封面 x 20个”）。
3. 保留完整的网址、@用户名、话题标签。
4. 如果数字是内容的一部分（例如提示词中的“20个”），则保留；若数字是孤立的点赞数、阅读数、时间戳，则删除。
【删除规则】
1. 删除所有连续重复的无意义单词，例如堆叠的“ALT”（只保留第一个有上下文的，或全部删除）。
2. 删除孤立的数字（如“5”“18”“93”“8,979”），除非它们明显是参数或列表项的一部分。
3. 删除时间戳（如“下午2:25 · 2026年5月6日”）、查看数、点赞/转发计数、平台按钮文字（如“查看”“相关”）。
4. 删除广告或无关引导（如“想发布自己的文章？升级为 Premium”）。
5. 删除无意义的单个字符、表情符号（除非有语义）、重复的分隔线。；
要求二：有英文内容的，在下面一段插入对应的中文翻译；
要求三：重新整理排版内容，使得文字阅读性强，但是不能更改原文内容，特别是笔记里面提到的类似于“提示词”之类的绝对不能更改（只允许要求二的插入翻译内容）；
要求四：把每个笔记的标题都改成能够体现对应笔记内容的标题。
要求五：最后完成所有内容后，把文件名改成对应的内容的标题的名称。

```









