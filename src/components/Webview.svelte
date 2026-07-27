<!--
 Copyright (C) 2023 Zuoqiu Yingyi

 This program is free software: you can redistribute it and/or modify
 it under the terms of the GNU Affero General Public License as
 published by the Free Software Foundation, either version 3 of the
 License, or (at your option) any later version.

 This program is distributed in the hope that it will be useful,
 but WITHOUT ANY WARRANTY; without even the implied warranty of
 MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 GNU Affero General Public License for more details.

 You should have received a copy of the GNU Affero General Public License
 along with this program.  If not, see <http://www.gnu.org/licenses/>.
-->

<script lang="ts">
    import { onMount } from "svelte";
    import { fade } from "svelte/transition";

    import BlockIcon from "@workspace/components/siyuan/misc/BlockIcon.svelte";
    import { TooltipsDirection } from "@workspace/components/siyuan/misc/tooltips";
    import Tab from "@workspace/components/siyuan/tab/Tab.svelte";
    import { nativeImage } from "@workspace/utils/electron";
    import clipboard from "@workspace/utils/electron/clipboard";
    import { FLAG_ELECTRON } from "@workspace/utils/env/native-front-end";
    import { base64ToDataURL } from "@workspace/utils/misc/dataurl";
    import { escapeHTML } from "@workspace/utils/misc/html";
    import { trimPrefix } from "@workspace/utils/misc/string";
    import { washMenuItems } from "@workspace/utils/siyuan/menu/wash";
    import { isStaticPathname } from "@workspace/utils/siyuan/url";

    import type siyuan from "siyuan";

    import type { Electron } from "@workspace/types/electron";

    import type WebviewPlugin from "@/index";

    interface IProps {
        src: string;
        tab: siyuan.Custom;
        title: string;
        plugin: InstanceType<typeof WebviewPlugin>;
    }

    let { src, tab, title = "", plugin }: IProps = $props();

    // svelte-ignore state_referenced_locally
    const useragent: string = plugin.useragent; // 用户代理
    // svelte-ignore state_referenced_locally
    const background: string = plugin.background; // 背景
    // svelte-ignore state_referenced_locally
    const i18n = plugin.i18n;

    let menu: InstanceType<typeof plugin.siyuan.Menu> | undefined;

    let fullscreen = $state(false); // 是否为全屏模式
    let can_back = $state(false); // 能否转到上一页
    let can_forward = $state(false); // 能否转到下一页
    let loading = $state(false); // 页面是否正在加载
    // svelte-ignore state_referenced_locally
    let address = $state(globalThis.decodeURIComponent(src)); // 地址栏
    // svelte-ignore state_referenced_locally
    let href = $state(src); // 当前页面链接
    let devtools_opened = $state(false); // 开发者工具是否已打开

    let iframe: HTMLIFrameElement | null = $state(null); // iframe 标签
    let webview: Electron.WebviewTag | undefined = $state(undefined); // webview 标签
    let webview_pointer_events_disable = $state(false); // 是否禁用 webview 的鼠标事件

    let more: BlockIcon | undefined; // 更多按钮

    let mask: HTMLDivElement; // 遮罩
    let mask_active = $state(false); // 是否激活遮罩

    let status_display = $state(false); // 状态栏显示状态
    let status = $state(""); // 状态栏内容

    // svelte-ignore state_referenced_locally
    void iframe;

    /* 加载 URL */
    function loadURL(href: string): void {
        if (FLAG_ELECTRON) {
            try {
                webview?.loadURL?.(href, {
                    userAgent: useragent,
                });
            }
            catch (error) {
                void error;
                src = href;
            }
        }
        else {
            src = href;
        }
    }

    /* 转到上一页 */
    function onGoBack() {
        if (can_back) {
            webview?.goBack?.();
        }
    }

    /* 转到下一页 */
    function onGoForward() {
        if (can_forward) {
            webview?.goForward?.();
        }
    }

    /* 刷新或终止加载按钮 */
    function onRefreshOrStop() {
        if (loading) {
            webview?.stop?.();
        }
        else {
            webview?.reload?.();
        }
    }

    /* 地址栏存在来自外部更改 */
    function onAddressChange(_e: Event) {
        // plugin.logger.debug(e);

        if (address) {
            try {
                try {
                    // 判断是否为标准 URL
                    const url = new URL(address);
                    href = url.href;
                }
                catch (error) {
                    void error;
                    switch (true) {
                        case address.startsWith("//"): {
                            /* `//` 协议 */
                            const url = new URL(`https:${address}`);
                            href = url.href;
                            break;
                        }
                        case isStaticPathname(address, false): {
                            /* 是否为思源静态文件服务 */
                            const url = new URL(`${globalThis.document.baseURI}${trimPrefix(address, "/")}`);
                            href = url.href;
                            break;
                        }
                        default: {
                            /* 未设置协议的 URL */
                            const url = new URL(`https://${address}`);
                            href = url.href;
                            break;
                        }
                    }
                }
                loadURL(href);
            }
            catch (error) {
                plugin.logger.warn(error);
                plugin.siyuan.showMessage(`${plugin.name}:\nURL <code class="fn__code">${address}</code> ${i18n.message.nonStandardURL}\n`, undefined, "error");
            }
        }
    }

    /* 使用默认程序打开 */
    function onOpenWithDefaultProgram() {
        globalThis.open(tab.data.href, "_blank");
    }

    /* 在新窗口打开 */
    function onOpenWithNewWindow(e: MouseEvent) {
        plugin.openWindow(tab.data.href, {
            x: e.screenX,
            y: e.screenY,
            title: tab.data.title,
        });
    }

    /* 进入/退出全屏模式 */
    function onEnterOrExitFullscreen() {
        fullscreen = !fullscreen;
    }

    /* 打开/关闭开发者工具 */
    function onOpenOrCloseDevTools() {
        if (webview) {
            if (webview?.isDevToolsOpened?.()) {
                webview?.closeDevTools?.();
            }
            else {
                webview?.openDevTools?.();
            }
        }
    }

    /**
     * 构建 Markdown 链接
     * @param text - 链接文本
     * @param url - 链接地址
     * @param title - 链接标题
     */
    function buildMarkdownLink(text: string | undefined, url: string, title?: string): string {
        text = text || "🔗";
        const markdown: string[] = [];
        markdown.push("[");
        markdown.push(text.replaceAll("]", "\\]").replaceAll("\n", ""));
        markdown.push("](");
        markdown.push(url);
        if (title) {
            markdown.push(` "${title.replaceAll("\n", "").replaceAll("&", "&amp;").replaceAll("\"", "&quot;")}"`);
        }
        markdown.push(")");
        return markdown.join("");
    }

    /* 打开更多菜单 */
    function onOpenMoreMenu(e: MouseEvent) {
        setTimeout(() => {
            // 需要延迟执行，否则会被 mask 的点击事件阻止

            const items: siyuan.IMenu[] = [
                {
                    // 复制页面标题
                    icon: "icon-webview-title",
                    label: i18n.menu.copyTitle.label,
                    click: () => clipboard.writeText(title),
                },
                {
                    // 复制页面地址
                    icon: "iconLink",
                    label: i18n.menu.copyPageAddress.label,
                    accelerator: "URL",
                    click: () => clipboard.writeText(href),
                },
                {
                    type: "separator",
                },
                {
                    // 复制页面链接 (富文本)
                    icon: "iconLink",
                    label: i18n.menu.copyLink.label,
                    accelerator: escapeHTML("<a>"),
                    click: () => {
                        const a = globalThis.document.createElement("a");
                        a.href = href;
                        a.textContent = title;
                        clipboard.writeHTML(a.outerHTML);
                    },
                },
                {
                    // 复制链接 (HTML)
                    icon: "iconHTML5",
                    label: i18n.menu.copyLink.label,
                    accelerator: "HTML",
                    click: () => {
                        const a = globalThis.document.createElement("a");
                        a.href = href;
                        a.textContent = title;
                        clipboard.writeText(a.outerHTML);
                    },
                },
                {
                    // 复制链接 (Markdown)
                    icon: "iconMarkdown",
                    label: i18n.menu.copyLink.label,
                    accelerator: "Markdown",
                    click: () => {
                        clipboard.writeText(
                            buildMarkdownLink(
                                title,
                                href,
                            ),
                        );
                    },
                },
                {
                    type: "separator",
                },
                {
                    // 使用默认程序(一般为浏览器)打开当前页面链接
                    icon: "iconLanguage",
                    label: i18n.webview.openWithDefaultProgram,
                    click: () => onOpenWithDefaultProgram(),
                },
                {
                    // 使用新窗口打开当前页面链接
                    icon: "iconOpenWindow",
                    label: i18n.webview.openWithNewWindow,
                    click: (_element, event) => onOpenWithNewWindow(event),
                },
                {
                    type: "separator",
                },
                {
                    // 进入/退出全屏模式
                    icon: fullscreen ? "iconFullscreenExit" : "iconFullscreen",
                    // label: fullscreen ? i18n.webview.exitFullscreen : i18n.webview.enterFullscreen,
                    label: i18n.webview.fullscreenMode,
                    checked: fullscreen,
                    click: () => onEnterOrExitFullscreen(),
                },
                {
                    // 打开/关闭开发者工具
                    icon: "iconBug",
                    // label: devtools_opened ? i18n.webview.closeDevTools : i18n.webview.openDevTools,
                    label: i18n.webview.developerTools,
                    checked: devtools_opened,
                    click: () => onOpenOrCloseDevTools(),
                },
            ];

            menu = new plugin.siyuan.Menu(`${plugin.name}-more-menu`, () => {
                mask_active = false;
            });

            items.forEach((item) => menu?.addItem(item));

            mask_active = true;
            mask?.focus();

            const rect = more?.rect();
            menu?.open({
                x: rect?.right ?? e.pageX,
                y: rect?.bottom ?? e.pageY,
                isLeft: true,
            });
        }, 0);
    }

    onMount(() => {
        /**
         * 监听页面变化
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#event-load-commit
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#event-will-navigate
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#event-did-start-navigation
         */
        webview?.addEventListener?.("load-commit", (e) => {
            // plugin.logger.debug(e)
            /* 更新地址栏地址 */
            if (e.isMainFrame) {
                href = e.url;
                address = globalThis.decodeURIComponent(e.url);
                tab.data.href = e.url;
            }

            /* 是否可后退 */
            // REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#webviewcangoback
            can_back = webview?.canGoBack?.() ?? false;

            /* 是否可前进 */
            // REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#webviewcangoback
            can_forward = webview?.canGoForward?.() ?? false;
        });

        /**
         * 更改页签标题
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#%E4%BA%8B%E4%BB%B6-page-title-updated
         */
        webview?.addEventListener?.("page-title-updated", (e) => {
            // plugin.logger.debug(e)
            // plugin.logger.debug(tab);
            title = e.title;

            tab.data.title = title;
            tab.tab.updateTitle(title);
            tab.tab.headElement.ariaLabel = title;
        });

        /**
         * 更改页签图标
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#%E4%BA%8B%E4%BB%B6-page-favicon-updated
         */
        webview?.addEventListener?.("page-favicon-updated", (e) => {
            // plugin.logger.debug(e)
            const favicons = e.favicons;

            /* 删除原生 svg 图标 */
            tab.tab.headElement.querySelector(".item__graphic")?.remove();

            if (favicons.length > 0) {
                const favicon = favicons[0]!; // 图标地址
                const iconElement = tab.tab.headElement.querySelector(".item__icon"); // 图标容器

                /* 图标容器不存在或者图标地址更改时插入/更新图标 */
                if (tab.tab.docIcon !== favicon || !iconElement) {
                    tab.tab.docIcon = favicon;
                    const img = `<img src="${favicon}" />`; // 在线图标

                    /* 设置图标 */
                    if (iconElement) {
                        // 更新图标
                        iconElement.innerHTML = img;
                    }
                    else {
                        // 插入图标
                        tab.tab.headElement.insertAdjacentHTML("afterbegin", `<span class="item__icon">${img}</span>`);
                    }
                }
            }
            else {
                /* 设置默认图标 */
                tab.tab.setDocIcon("🌐".codePointAt(0)!.toString(16));
            }
        });

        /**
         * 加载时 & 加载完成设置不同的状态
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#event-did-start-loading
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#event-did-stop-loading
         */
        /* 开始加载 */
        webview?.addEventListener?.("did-start-loading", (_) => {
            // plugin.logger.debug(e)
            loading = true;
        });
        /* 停止加载 */
        webview?.addEventListener?.("did-stop-loading", (_) => {
            // plugin.logger.debug(e)
            loading = false;
        });

        /**
         * 开发者工具中打开超链接
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#event-devtools-open-url
         */
        webview?.addEventListener?.("devtools-open-url", (e) => {
            // plugin.logger.debug(e);
            plugin.openWebviewTab(e.url);
        });

        /**
         * 开启/关闭开发者工具
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#event-devtools-opened
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#event-devtools-closed
         */
        webview?.addEventListener?.("devtools-opened", (_e) => (devtools_opened = true));
        webview?.addEventListener?.("devtools-closed", (_e) => (devtools_opened = false));

        /**
         * 焦点为链接时在状态栏显示链接
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#event-update-target-url
         */
        webview?.addEventListener?.("update-target-url", (e) => {
            // plugin.logger.debug(e);

            if (e.url) {
                status = globalThis.decodeURIComponent(e.url);
                if (!status_display) {
                    status_display = true;
                }
            }
            else {
                status_display = false;
            }
        });

        /**
         * 上下文菜单(右键触发)
         * REF https://www.electronjs.org/zh/docs/latest/api/webview-tag#event-context-menu
         */
        webview?.addEventListener?.("context-menu", (e) => {
            plugin.logger.debug(e);
            const { params } = e;
            const title = params.titleText || params.linkText || params.altText || params.suggestedFilename;

            // 添加右键菜单
            const items: siyuan.IMenu[] = [];

            function buildOpenMenuItems(url: string, title: string, action: string, current: boolean = true): siyuan.IMenu[] {
                const items: siyuan.IMenu[] = [];

                if (current) {
                    /* 在当前页签中打开 */
                    items.push({
                        icon: "icon-webview-click",
                        label: i18n.menu.openTabCurrent.label,
                        action,
                        click: () => loadURL(url),
                    });

                    items.push({ type: "separator" });
                }

                /* 在新页签中打开 */
                items.push({
                    icon: "iconAdd",
                    label: i18n.menu.openTab.label,
                    action,
                    click: () => plugin.openWebviewTab(url, title),
                });

                /* 在后台页签中打开 */
                items.push({
                    icon: "iconMin",
                    label: i18n.menu.openTabBackground.label,
                    action,
                    click: () => plugin.openWebviewTab(url, title, undefined, { keepCursor: true }),
                });

                /* 在页签右侧打开 */
                items.push({
                    icon: "iconLayoutRight",
                    label: i18n.menu.openTabRight.label,
                    action,
                    click: () => plugin.openWebviewTab(url, title, undefined, { position: "right" }),
                });

                /* 在页签下侧打开 */
                items.push({
                    icon: "iconLayoutBottom",
                    label: i18n.menu.openTabBottom.label,
                    action,
                    click: () => plugin.openWebviewTab(url, title, undefined, { position: "bottom" }),
                });

                items.push({ type: "separator" });

                /* 在新窗口打开 */
                items.push({
                    icon: "iconOpenWindow",
                    label: i18n.menu.openByNewWindow.label,
                    action,
                    click: (_element, event) =>
                        void plugin.openWebpageWindow(url, title, {
                            screenX: event.screenX,
                            screenY: event.screenY,
                        }),
                });

                return items;
            }

            function buildCopyMenuItems(params: Electron.Params): siyuan.IMenu[] {
                const items: siyuan.IMenu[] = [];

                /* 复制链接地址 */
                if (params.linkURL) {
                    items.push({
                        icon: "iconLink",
                        label: i18n.menu.copyLinkAddress.label,
                        action: "iconLink",
                        click: () => clipboard.writeText(params.linkURL),
                    });
                }

                /* 复制资源地址 */
                if (params.srcURL) {
                    items.push({
                        icon: "iconLink",
                        label: i18n.menu.copyResourceAddress.label,
                        action: "iconCloud",
                        click: () => clipboard.writeText(params.srcURL),
                    });
                }

                /* 复制框架地址 */
                if (params.frameURL) {
                    items.push({
                        icon: "iconLink",
                        label: i18n.menu.copyFrameAddress.label,
                        action: "iconLayout",
                        click: () => clipboard.writeText(params.frameURL),
                    });
                }

                /* 复制页面地址 */
                if (params.pageURL) {
                    items.push({
                        icon: "iconLink",
                        label: i18n.menu.copyPageAddress.label,
                        action: "iconFile",
                        click: () => clipboard.writeText(params.pageURL),
                    });
                }

                items.push({ type: "separator" });

                /* 复制标题 */
                if (params.titleText) {
                    items.push({
                        icon: "icon-webview-title",
                        label: i18n.menu.copyTitle.label,
                        click: () => clipboard.writeText(params.titleText),
                    });
                }

                /* 复制描述 */
                if (params.altText) {
                    items.push({
                        icon: "iconInfo",
                        label: i18n.menu.copyAlt.label,
                        click: () => clipboard.writeText(params.altText),
                    });
                }

                /* 复制锚文本 */
                if (params.linkText) {
                    items.push({
                        icon: "icon-webview-anchor",
                        label: i18n.menu.copyText.label,
                        click: () => clipboard.writeText(params.linkText),
                    });
                }

                /* 复制文件名 */
                if (params.suggestedFilename) {
                    items.push({
                        icon: "iconN",
                        label: i18n.menu.copyFileName.label,
                        click: () => clipboard.writeText(params.suggestedFilename),
                    });
                }

                return items;
            }

            function getValidTexts(...args: string[]): string[] {
                return args.filter((text) => !!text);
            }

            /* 复制划选内容 */
            if (params.selectionText) {
                items.push({
                    icon: "icon-webview-select",
                    label: i18n.menu.copySelectionText.label,
                    click: () => clipboard.writeText(params.selectionText),
                });
                items.push({ type: "separator" });
            }

            switch (params.mediaType) {
                case "none":
                case "file":
                case "canvas":
                case "plugin":
                // eslint-disable-next-line default-case-last, no-fallthrough
                default: {
                    switch (true) {
                        case !!params.linkURL: {
                            items.push(...buildOpenMenuItems(params.linkURL, title, "iconLink"));

                            items.push({ type: "separator" });

                            /* 复制链接 (富文本) */
                            items.push({
                                icon: "iconLink",
                                label: i18n.menu.copyLink.label,
                                accelerator: escapeHTML("<a>"),
                                click: () => {
                                    const a = globalThis.document.createElement("a");
                                    a.href = params.linkURL;
                                    a.title = params.titleText;
                                    a.textContent = params.linkText;
                                    clipboard.writeHTML(a.outerHTML);
                                },
                            });

                            /* 复制链接 (HTML) */
                            items.push({
                                icon: "iconHTML5",
                                label: i18n.menu.copyLink.label,
                                accelerator: "HTML",
                                click: () => {
                                    const a = globalThis.document.createElement("a");
                                    a.href = params.linkURL;
                                    a.title = params.titleText;
                                    a.textContent = params.linkText;
                                    clipboard.writeText(a.outerHTML);
                                },
                            });

                            /* 复制链接 (Markdown) */
                            items.push({
                                icon: "iconMarkdown",
                                label: i18n.menu.copyLink.label,
                                accelerator: "Markdown",
                                click: () => {
                                    const texts = getValidTexts(params.linkText, params.altText, params.suggestedFilename, params.titleText);
                                    clipboard.writeText(
                                        buildMarkdownLink(
                                            texts.shift(), //
                                            params.linkURL, //
                                            texts.pop(), //
                                        ),
                                    );
                                },
                            });
                            break;
                        }
                        case !!params.frameURL: {
                            items.push(...buildOpenMenuItems(params.frameURL, title, "iconLayout"));

                            items.push({ type: "separator" });

                            /* 复制框架 (富文本) */
                            items.push({
                                icon: "iconLayout",
                                label: i18n.menu.copyFrame.label,
                                accelerator: escapeHTML("<iframe>"),
                                click: () => {
                                    const iframe = globalThis.document.createElement("iframe");
                                    iframe.src = params.frameURL;
                                    iframe.title = params.titleText;
                                    clipboard.writeHTML(iframe.outerHTML);
                                },
                            });

                            /* 复制框架 (HTML) */
                            items.push({
                                icon: "iconHTML5",
                                label: i18n.menu.copyFrame.label,
                                accelerator: "HTML",
                                click: () => {
                                    const iframe = globalThis.document.createElement("iframe");
                                    iframe.src = params.frameURL;
                                    iframe.title = params.titleText;
                                    clipboard.writeText(iframe.outerHTML);
                                },
                            });

                            /* 复制框架 (Markdown) */
                            items.push({
                                icon: "iconMarkdown",
                                label: i18n.menu.copyFrame.label,
                                accelerator: "Markdown",
                                click: () => {
                                    const texts = getValidTexts(
                                        params.linkText, //
                                        params.altText, //
                                        params.suggestedFilename, //
                                        params.titleText, //
                                    );
                                    clipboard.writeText(
                                        buildMarkdownLink(
                                            texts.shift(), //
                                            params.frameURL, //
                                            texts.pop(), //
                                        ),
                                    );
                                },
                            });
                            break;
                        }
                        default: {
                            items.push(...buildOpenMenuItems(params.pageURL, title, "iconFile", false));

                            items.push({ type: "separator" });

                            /* 复制页面链接 (富文本) */
                            items.push({
                                icon: "iconFile",
                                label: i18n.menu.copyPage.label,
                                accelerator: escapeHTML("<a>"),
                                click: () => {
                                    const a = globalThis.document.createElement("a");
                                    a.href = params.pageURL;
                                    a.title = params.titleText;
                                    clipboard.writeHTML(a.outerHTML);
                                },
                            });

                            /* 复制页面链接 (HTML) */
                            items.push({
                                icon: "iconHTML5",
                                label: i18n.menu.copyPage.label,
                                accelerator: "HTML",
                                click: () => {
                                    const a = globalThis.document.createElement("a");
                                    a.href = params.pageURL;
                                    a.title = params.titleText;
                                    clipboard.writeText(a.outerHTML);
                                },
                            });

                            /* 复制页面链接 (Markdown) */
                            items.push({
                                icon: "iconMarkdown",
                                label: i18n.menu.copyPage.label,
                                accelerator: "Markdown",
                                click: () => {
                                    const texts = getValidTexts(
                                        params.linkText, //
                                        params.altText, //
                                        params.suggestedFilename, //
                                        params.titleText, //
                                    );
                                    clipboard.writeText(
                                        buildMarkdownLink(
                                            texts.shift(), //
                                            params.pageURL, //
                                            texts.pop(), //
                                        ),
                                    );
                                },
                            });
                            break;
                        }
                    }
                    break;
                }

                /* 图片 */
                case "image": {
                    items.push(...buildOpenMenuItems(params.linkURL, title, "iconImage"));

                    items.push({ type: "separator" });

                    /* 复制图片 (图片文件) */
                    items.push({
                        icon: "iconImage",
                        label: i18n.menu.copyImage.label,
                        click: () => {
                            setTimeout(async () => {
                                try {
                                    const response = await plugin.client.forwardProxy({
                                        headers: [],
                                        method: "GET",
                                        responseEncoding: "base64",
                                        timeout: 60_000,
                                        url: params.srcURL,
                                    });
                                    if (response.data.status >= 200 && response.data.status < 300) {
                                        const data_url = base64ToDataURL(response.data.body, response.data.contentType);
                                        const image = nativeImage.createFromDataURL(data_url);
                                        clipboard.writeImage(image);
                                    }
                                }
                                catch (error) {
                                    plugin.logger.warn(error);
                                }
                                finally {
                                    menu?.close();
                                }
                            });
                            return true;
                        },
                    });

                    /* 复制图片 (富文本) */
                    items.push({
                        icon: "iconImage",
                        label: i18n.menu.copyImage.label,
                        accelerator: escapeHTML("<img>"),
                        click: () => {
                            const img = globalThis.document.createElement("img");
                            img.src = params.srcURL;
                            img.title = params.titleText;
                            img.alt = params.altText;
                            clipboard.writeHTML(img.outerHTML);
                        },
                    });

                    /* 复制图片 (HTML) */
                    items.push({
                        icon: "iconHTML5",
                        label: i18n.menu.copyImage.label,
                        accelerator: "HTML",
                        click: () => {
                            const img = globalThis.document.createElement("img");
                            img.src = params.srcURL;
                            img.title = params.titleText;
                            img.alt = params.altText;
                            clipboard.writeText(img.outerHTML);
                        },
                    });

                    /* 复制图片 (Markdown) */
                    items.push({
                        icon: "iconMarkdown",
                        label: i18n.menu.copyImage.label,
                        accelerator: "Markdown",
                        click: () => {
                            const texts = getValidTexts(
                                params.altText, //
                                params.linkText, //
                                params.suggestedFilename, //
                                params.titleText, //
                            );
                            clipboard.writeText(
                                buildMarkdownLink(
                                    texts.shift(), //
                                    params.srcURL, //
                                    texts.pop(), //
                                ),
                            );
                        },
                    });
                    break;
                }

                /* 音频 */
                case "audio": {
                    items.push(...buildOpenMenuItems(params.srcURL, title, "iconRecord"));

                    items.push({ type: "separator" });

                    /* 复制音频 (富文本) */
                    items.push({
                        icon: "iconRecord",
                        label: i18n.menu.copyAudio.label,
                        accelerator: escapeHTML("<audio>"),
                        click: () => {
                            const audio = globalThis.document.createElement("audio");
                            audio.src = params.srcURL;
                            audio.title = params.titleText;
                            clipboard.writeHTML(audio.outerHTML);
                        },
                    });

                    /* 复制音频 (HTML) */
                    items.push({
                        icon: "iconHTML5",
                        label: i18n.menu.copyAudio.label,
                        accelerator: "HTML",
                        click: () => {
                            const audio = globalThis.document.createElement("audio");
                            audio.src = params.srcURL;
                            audio.title = params.titleText;
                            clipboard.writeText(audio.outerHTML);
                        },
                    });

                    /* 复制音频 (Markdown) */
                    items.push({
                        icon: "iconMarkdown",
                        label: i18n.menu.copyAudio.label,
                        accelerator: "Markdown",
                        click: () => {
                            const texts = getValidTexts(
                                params.altText, //
                                params.linkText, //
                                params.suggestedFilename, //
                                params.titleText, //
                            );
                            clipboard.writeText(
                                buildMarkdownLink(
                                    texts.shift(), //
                                    params.srcURL, //
                                    texts.pop(), //
                                ),
                            );
                        },
                    });
                    break;
                }

                /* 视频 */
                case "video": {
                    items.push(...buildOpenMenuItems(params.srcURL, title, "iconVideo"));

                    items.push({ type: "separator" });

                    /* 复制视频 (富文本) */
                    items.push({
                        icon: "iconVideo",
                        label: i18n.menu.copyVideo.label,
                        accelerator: escapeHTML("<video>"),
                        click: () => {
                            const video = globalThis.document.createElement("video");
                            video.src = params.srcURL;
                            video.title = params.titleText;
                            clipboard.writeHTML(video.outerHTML);
                        },
                    });

                    /* 复制视频 (HTML) */
                    items.push({
                        icon: "iconHTML5",
                        label: i18n.menu.copyVideo.label,
                        accelerator: "HTML",
                        click: () => {
                            const video = globalThis.document.createElement("video");
                            video.src = params.srcURL;
                            video.title = params.titleText;
                            clipboard.writeText(video.outerHTML);
                        },
                    });

                    /* 复制视频 (Markdown) */
                    items.push({
                        icon: "iconMarkdown",
                        label: i18n.menu.copyVideo.label,
                        accelerator: "Markdown",
                        click: () => {
                            const texts = getValidTexts(
                                params.altText, //
                                params.linkText, //
                                params.suggestedFilename, //
                                params.titleText, //
                            );
                            clipboard.writeText(
                                buildMarkdownLink(
                                    texts.shift(), //
                                    params.srcURL, //
                                    texts.pop(), //
                                ),
                            );
                        },
                    });
                    break;
                }
            }

            /* 复制指定字段 */
            items.push({ type: "separator" });
            items.push(...buildCopyMenuItems(params));

            const _items = washMenuItems(items);
            if (_items.length > 0) {
                menu = new plugin.siyuan.Menu("plugin-webview-menu", () => {
                    mask_active = false;
                });
                _items.forEach((item) => menu?.addItem(item));

                mask_active = true;
                mask?.focus();
                menu?.open({
                    x: params.x,
                    y: params.y,
                });
            }
        });
    });

    function onmouseenter(e: MouseEvent): void {
        e.stopPropagation();
        webview_pointer_events_disable = e.button !== 0;
    }
    function onmouseleave(e: MouseEvent): void {
        e.stopPropagation();
        webview_pointer_events_disable = true;
    }
    function onMaskClick(_e: MouseEvent): void {
        menu?.close?.();
    }
</script>

<Tab {fullscreen}>
    <!-- 地址栏 -->
    <div
        slot="breadcrumb"
        class="protyle-breadcrumb"
    >
        <!-- 后退按钮 -->
        <BlockIcon
            ariaLabel={i18n.webview.goForwardOnePage}
            disabled={!can_back}
            icon="#iconLeft"
            tooltipsDirection={TooltipsDirection.se}
            on:click={onGoBack}
        />

        <!-- 前进按钮 -->
        <BlockIcon
            ariaLabel={i18n.webview.goBackOnePage}
            disabled={!can_forward}
            icon="#iconRight"
            tooltipsDirection={TooltipsDirection.se}
            on:click={onGoForward}
        />

        <!-- 刷新/终止加载按钮 -->
        <BlockIcon
            ariaLabel={loading ? i18n.webview.stopLoadingThisPage : i18n.webview.reloadCurrentPage}
            icon={loading ? "#iconClose" : "#iconRefresh"}
            tooltipsDirection={TooltipsDirection.se}
            on:click={onRefreshOrStop}
        />

        <!-- <div class="fn__space" /> -->

        <!-- 地址输入框 -->
        <input
            class="b3-text-field fn__flex-1 address-field"
            onchange={onAddressChange}
            type="url"
            bind:value={address}
        />

        <!-- <div class="fn__space" /> -->

        <!-- 更多按钮 -->
        <BlockIcon
            bind:this={more}
            ariaLabel={i18n.webview.more}
            icon="#iconMore"
            tooltipsDirection={TooltipsDirection.sw}
            on:click={onOpenMoreMenu}
        />
    </div>

    <!-- 主体 -->
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <!-- svelte-ignore a11y_click_events_have_key_events -->
    <div
        slot="content"
        class="content fn__flex fn__flex-1"
        {onmouseenter}
        {onmouseleave}
    >
        {#if FLAG_ELECTRON}
            <webview
                bind:this={webview}
                style:background
                class="webview fn__flex-1"
                class:pointer-events-disable={webview_pointer_events_disable}
                allowpopups
                {src}
                {title}
                {useragent}
            ></webview>
        {:else}
            <iframe
                bind:this={iframe}
                style:background
                class="fn__flex-1"
                allowfullscreen
                {src}
                {title}
            ></iframe>
        {/if}
        {#if status_display}
            <!-- 状态提示 (显示超链接地址) -->
            <div
                class="webview-status tooltip"
                in:fade={{ delay: 0, duration: 125 }}
                out:fade={{ delay: 500, duration: 250 }}
            >
                <span>{status}</span>
            </div>
        {/if}
        <!-- 右键菜单遮罩 (点击后关闭菜单) -->
        <div
            bind:this={mask}
            class="mask"
            class:mask-active={mask_active}
            onclick={onMaskClick}
        ></div>
    </div>
</Tab>

<style lang="less">
    .protyle-breadcrumb {
        height: 32px;

        .address-field {
            margin: 4px;
        }
    }

    // .protyle-preview {
    //     user-select: none;
    // }
    .content {
        user-select: none;
        position: relative;
    }

    .mask {
        display: none;

        &.mask-active {
            display: block;
            position: absolute;
            height: 100%;
            width: 100%;
        }
    }

    .webview-status {
        position: absolute;
        bottom: 0;
        left: 0;
        z-index: 1;
    }

    .pointer-events-disable {
        pointer-events: none;
    }
</style>
