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

<!-- 设置面板 -->

<script lang="ts">
    import Group from "@workspace/components/siyuan/setting/item/Group.svelte";
    import Input from "@workspace/components/siyuan/setting/item/Input.svelte";
    import { ItemType } from "@workspace/components/siyuan/setting/item/item";
    import Item from "@workspace/components/siyuan/setting/item/Item.svelte";
    import MiniItem from "@workspace/components/siyuan/setting/item/MiniItem.svelte";
    import Panel from "@workspace/components/siyuan/setting/panel/Panel.svelte";
    import Panels from "@workspace/components/siyuan/setting/panel/Panels.svelte";
    import Shortcut from "@workspace/components/siyuan/setting/specified/Shortcut.svelte";
    import Tabs from "@workspace/components/siyuan/setting/tab/Tabs.svelte";
    import { MouseButton } from "@workspace/utils/shortcut";
    import { EditorType } from "@workspace/utils/siyuan";

    import { MenuBarStatus } from "@/utils/window";

    import type { ITab } from "@workspace/components/siyuan/setting/tab";

    import type WebviewPlugin from "@/index";
    import type { IConfig } from "@/types/config";

    interface IProps {
        config: IConfig; // 传入的配置项
        plugin: InstanceType<typeof WebviewPlugin>; // 插件实例
    }

    const {
        config,
        plugin,
    }: IProps = $props();

    const i18n = plugin.i18n;

    function updated() {
        plugin.updateConfig(config);
    }

    function resetOptions() {
        plugin.siyuan.confirm(
            i18n.settings.generalSettings.reset.title, // 标题
            i18n.settings.generalSettings.reset.description, // 文本
            async () => {
                await plugin.resetConfig(); // 重置配置
                globalThis.location.reload(); // 刷新页面
            }, // 确认按钮回调
        );
    }

    const PanelKey = {
        general: "general",
        openTab: "openTab",
        openWindow: "openWindow",
    } as const;

    const TabKey = {
        general: "general",
        hyperlink: "hyperlink",
        shortcut: "shortcut",
        siyuan: "siyuan",
    } as const;

    const panels_focus_key = PanelKey.general;
    const panels = [
        {
            key: PanelKey.general,
            text: i18n.settings.generalSettings.title,
            name: i18n.settings.generalSettings.title,
            icon: "#iconSettings",
        },
        {
            key: PanelKey.openTab,
            text: i18n.settings.openTab,
            name: i18n.settings.openTab,
            icon: "#iconBoth",
        },
        {
            key: PanelKey.openWindow,
            text: i18n.settings.openWindow,
            name: i18n.settings.openWindow,
            icon: "#iconOpenWindow",
        },
    ] as const satisfies ITab[];

    const tab_settings_tabs_focus_key = TabKey.general;
    const window_settings_tabs_focus_key = TabKey.general;
    const tabs = {
        tab: [
            {
                key: TabKey.general,
                text: i18n.settings.general,
                name: i18n.settings.general,
                icon: "⚙",
            },
            {
                key: TabKey.hyperlink,
                text: i18n.settings.hyperlink,
                name: i18n.settings.hyperlink,
                icon: "🌐",
            },
            {
                key: TabKey.shortcut,
                text: i18n.settings.shortcut,
                name: i18n.settings.shortcut,
                icon: "⌨",
            },
        ] as const satisfies ITab[],
        window: [
            {
                key: TabKey.general,
                text: i18n.settings.general,
                name: i18n.settings.general,
                icon: "⚙",
            },
            {
                key: TabKey.hyperlink,
                text: i18n.settings.hyperlink,
                name: i18n.settings.hyperlink,
                icon: "🌐",
            },
            {
                key: TabKey.shortcut,
                text: i18n.settings.shortcut,
                name: i18n.settings.shortcut,
                icon: "⌨",
            },
            {
                key: TabKey.siyuan,
                text: i18n.settings.siyuan.title,
                name: i18n.settings.siyuan.title,
                icon: "📝",
            },
        ] as const satisfies ITab[],
    };
</script>

<Panels
    focus={panels_focus_key}
    {panels}
    let:focus={focusPanel}
>
    <!-- 常规设置面板 -->
    <Panel display={panels[0].key === focusPanel}>
        <!-- 自定义 UA -->
        <Item
            block={true}
            text={i18n.settings.generalSettings.useragent.description}
            title={i18n.settings.generalSettings.useragent.title}
        >
            <Input
                slot="input"
                block={true}
                placeholder={globalThis.navigator.userAgent}
                settingKey="Text"
                settingValue={config.general.useragent}
                type={ItemType.text}
                on:changed={(e) => {
                    config.general.useragent = e.detail.value;
                    updated();
                }}
            />
        </Item>

        <!-- 背景颜色 -->
        <Item
            text={i18n.settings.background.description}
            title={i18n.settings.background.title}
        >
            <Input
                slot="input"
                settingKey="text"
                settingValue={config.general.background}
                type={ItemType.text}
                on:changed={(e) => {
                    config.general.background = e.detail.value;
                    updated();
                }}
            />
        </Item>

        <!-- 重置设置 -->
        <Item
            text={i18n.settings.generalSettings.reset.description}
            title={i18n.settings.generalSettings.reset.title}
        >
            <Input
                slot="input"
                settingKey="Reset"
                settingValue={i18n.settings.generalSettings.reset.text}
                type={ItemType.button}
                on:clicked={resetOptions}
            />
        </Item>
    </Panel>

    <!-- 打开页签的设置面板 -->
    <Panel display={panels[1].key === focusPanel}>
        <Tabs
            focus={tab_settings_tabs_focus_key}
            tabs={tabs.tab}
            let:focus={focusTab}
        >
            <!-- 标签页 1 - 通用设置 -->
            <div
                class:fn__none={tabs.tab[0].key !== focusTab}
                data-type={tabs.tab[0].name}
            >
                <!-- 是否启用 -->
                <Item
                    text={i18n.settings.open.enable.tab.description}
                    title={i18n.settings.open.enable.tab.title}
                >
                    <Input
                        slot="input"
                        settingKey="Checkbox"
                        settingValue={config.tab.enable}
                        type={ItemType.checkbox}
                        on:changed={(e) => {
                            config.tab.enable = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 编辑器超链接 -->
                <Item
                    text={i18n.settings.open.editorHyperlink.description}
                    title={i18n.settings.open.editorHyperlink.title}
                >
                    <Input
                        slot="input"
                        settingKey="Checkbox"
                        settingValue={config.tab.open.targets.hyperlink.editor.enable}
                        type={ItemType.checkbox}
                        on:changed={(e) => {
                            config.tab.open.targets.hyperlink.editor.enable = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 其他超链接 -->
                <Item
                    text={i18n.settings.open.otherHyperlink.description}
                    title={i18n.settings.open.otherHyperlink.title}
                >
                    <Input
                        slot="input"
                        settingKey="Checkbox"
                        settingValue={config.tab.open.targets.hyperlink.other.enable}
                        type={ItemType.checkbox}
                        on:changed={(e) => {
                            config.tab.open.targets.hyperlink.other.enable = e.detail.value;
                            updated();
                        }}
                    />
                </Item>
            </div>
            <!-- 标签页 2 - 超链接设置 -->
            <div
                class:fn__none={tabs.tab[1].key !== focusTab}
                data-type={tabs.tab[1].name}
            >
                <!-- URL 协议 -->
                <Group title={i18n.settings.protocols.title}>
                    {#each Object.entries(config.tab.open.protocols) as [key, protocol] (key)}
                        <MiniItem minWidth="8em">
                            <code
                                slot="title"
                                class="fn__code">{protocol.prefix}</code
                            >
                            <Input
                                slot="input"
                                settingKey="Checkbox"
                                settingValue={protocol.enable}
                                type={ItemType.checkbox}
                                on:changed={(e) => {
                                    protocol.enable = e.detail.value;
                                    updated();
                                }}
                            />
                        </MiniItem>
                    {/each}
                </Group>

                <!-- 资源文件路径名 -->
                <Group title={i18n.settings.pathnames.title}>
                    {#each Object.entries(config.tab.open.pathnames) as [key, pathname] (key)}
                        <MiniItem
                            marginRight="1em"
                            minWidth="9em"
                        >
                            <code
                                slot="title"
                                class="fn__code">{pathname.prefix}</code
                            >
                            <Input
                                slot="input"
                                settingKey="Checkbox"
                                settingValue={pathname.enable}
                                type={ItemType.checkbox}
                                on:changed={(e) => {
                                    pathname.enable = e.detail.value;
                                    updated();
                                }}
                            />
                        </MiniItem>
                    {/each}
                </Group>
            </div>
            <!-- 标签页 3 - 快捷键设置 -->
            <div
                class:fn__none={tabs.tab[2].key !== focusTab}
                data-type={tabs.tab[2].name}
            >
                <Shortcut
                    disabledMouseButton={true}
                    displayMouseEvent={false}
                    minWidth="16em"
                    mouseButtonOptions={[{ key: MouseButton.Left, text: i18n.settings.mouse.left }]}
                    mouseButtonTitle={i18n.settings.mouse.button}
                    shortcut={config.tab.open.mouse}
                    title={i18n.settings.open.shortcut.title}
                    on:changed={updated}
                />
            </div>
        </Tabs>
    </Panel>

    <!-- 打开窗口的设置面板 -->
    <Panel display={panels[2].key === focusPanel}>
        <Tabs
            focus={window_settings_tabs_focus_key}
            tabs={tabs.window}
            let:focus={focusTab}
        >
            <!-- 标签页 1 - 通用设置 -->
            <div
                class:fn__none={tabs.window[0].key !== focusTab}
                data-type={tabs.window[0].name}
            >
                <!-- 是否启用 -->
                <Item
                    text={i18n.settings.open.enable.window.description}
                    title={i18n.settings.open.enable.window.title}
                >
                    <Input
                        slot="input"
                        settingKey="enable"
                        settingValue={config.window.enable}
                        type={ItemType.checkbox}
                        on:changed={(e) => {
                            config.window.enable = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 编辑器超链接 -->
                <Item
                    text={i18n.settings.open.editorHyperlink.description}
                    title={i18n.settings.open.editorHyperlink.title}
                >
                    <Input
                        slot="input"
                        settingKey="editorHyperlink"
                        settingValue={config.window.open.targets.hyperlink.editor}
                        type={ItemType.checkbox}
                        on:changed={(e) => {
                            config.window.open.targets.hyperlink.editor = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 其他超链接 -->
                <Item
                    text={i18n.settings.open.otherHyperlink.description}
                    title={i18n.settings.open.otherHyperlink.title}
                >
                    <Input
                        slot="input"
                        settingKey="otherHyperlink"
                        settingValue={config.window.open.targets.hyperlink.other}
                        type={ItemType.checkbox}
                        on:changed={(e) => {
                            config.window.open.targets.hyperlink.other.enable = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 窗口宽度 -->
                <Item
                    text={i18n.settings.window.width.description}
                    title={i18n.settings.window.width.title}
                >
                    <Input
                        slot="input"
                        limits={{ min: 320, max: 15360, step: 40 }}
                        settingKey="width"
                        settingValue={config.window.params.width}
                        type={ItemType.number}
                        on:changed={(e) => {
                            config.window.params.width = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 窗口高度 -->
                <Item
                    text={i18n.settings.window.height.description}
                    title={i18n.settings.window.height.title}
                >
                    <Input
                        slot="input"
                        limits={{ min: 240, max: 8640, step: 40 }}
                        settingKey="height"
                        settingValue={config.window.params.height}
                        type={ItemType.number}
                        on:changed={(e) => {
                            config.window.params.height = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 窗口居中 -->
                <Item
                    text={i18n.settings.window.center.description}
                    title={i18n.settings.window.center.title}
                >
                    <Input
                        slot="input"
                        settingKey="center"
                        settingValue={config.window.params.center}
                        type={ItemType.checkbox}
                        on:changed={(e) => {
                            config.window.params.center = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 窗口置顶 -->
                <Item
                    text={i18n.settings.window.top.description}
                    title={i18n.settings.window.top.title}
                >
                    <Input
                        slot="input"
                        settingKey="top"
                        settingValue={config.window.params.alwaysOnTop}
                        type={ItemType.checkbox}
                        on:changed={(e) => {
                            config.window.params.alwaysOnTop = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 窗口菜单栏 -->
                <Item
                    text={i18n.settings.window.menuBar.description}
                    title={i18n.settings.window.menuBar.title}
                >
                    <Input
                        slot="input"
                        options={[
                            { key: MenuBarStatus.AutoHide, text: i18n.settings.window.menuBar.options.autoHide },
                            { key: MenuBarStatus.AlwaysShow, text: i18n.settings.window.menuBar.options.alwaysShow },
                            { key: MenuBarStatus.Disabled, text: i18n.settings.window.menuBar.options.disabled },
                        ]}
                        settingKey="menuBar"
                        settingValue={config.window.params.enableMenuBar // 是否启用菜单栏
                            ? config.window.params.autoHideMenuBar // 是否自动隐藏菜单栏
                                ? MenuBarStatus.AutoHide // 自动隐藏
                                : MenuBarStatus.AlwaysShow // 总是显示
                            : MenuBarStatus.Disabled}
                        type={ItemType.select}
                        on:changed={(e) => {
                            switch (e.detail.value) {
                                case MenuBarStatus.AutoHide:
                                    config.window.params.enableMenuBar = true;
                                    config.window.params.autoHideMenuBar = true;
                                    break;
                                case MenuBarStatus.AlwaysShow:
                                    config.window.params.enableMenuBar = true;
                                    config.window.params.autoHideMenuBar = false;
                                    break;
                                case MenuBarStatus.Disabled:
                                    config.window.params.enableMenuBar = false;
                                    break;
                            }
                            updated();
                        }}
                    />
                </Item>
            </div>

            <!-- 标签页 2 - 超链接设置 -->
            <div
                class:fn__none={tabs.window[1].key !== focusTab}
                data-type={tabs.window[1].name}
            >
                <!-- URL 协议 -->
                <Group title={i18n.settings.protocols.title}>
                    {#each Object.entries(config.window.open.protocols) as [key, protocol] (key)}
                        <MiniItem minWidth="8em">
                            <code
                                slot="title"
                                class="fn__code">{protocol.prefix}</code
                            >
                            <Input
                                slot="input"
                                settingKey="Checkbox"
                                settingValue={protocol.enable}
                                type={ItemType.checkbox}
                                on:changed={(e) => {
                                    protocol.enable = e.detail.value;
                                    updated();
                                }}
                            />
                        </MiniItem>
                    {/each}
                </Group>

                <!-- 资源文件路径 -->
                <Group title={i18n.settings.pathnames.title}>
                    {#each Object.entries(config.window.open.pathnames) as [key, pathname] (key)}
                        <MiniItem
                            marginRight="1em"
                            minWidth="9em"
                        >
                            <code
                                slot="title"
                                class="fn__code">{pathname.prefix}</code
                            >
                            <Input
                                slot="input"
                                settingKey="Checkbox"
                                settingValue={pathname.enable}
                                type={ItemType.checkbox}
                                on:changed={(e) => {
                                    pathname.enable = e.detail.value;
                                    updated();
                                }}
                            />
                        </MiniItem>
                    {/each}
                </Group>
            </div>

            <!-- 标签页 3 - 快捷键设置 -->
            <div
                class:fn__none={tabs.window[2].key !== focusTab}
                data-type={tabs.window[2].name}
            >
                <Shortcut
                    disabledMouseButton={true}
                    displayMouseEvent={false}
                    minWidth="16em"
                    mouseButtonOptions={[{ key: MouseButton.Middle, text: i18n.settings.mouse.middle }]}
                    mouseButtonTitle={i18n.settings.mouse.button}
                    shortcut={config.window.open.mouse}
                    title={i18n.settings.open.shortcut.title}
                    on:changed={updated}
                />
            </div>

            <!-- 标签页 4 - 思源窗口设置 -->
            <div
                class:fn__none={tabs.window[3].key !== focusTab}
                data-type={tabs.window[3].name}
            >
                <!-- 是否启用 -->
                <Item
                    text={i18n.settings.siyuan.enable.description}
                    title={i18n.settings.siyuan.enable.title}
                >
                    <Input
                        slot="input"
                        settingKey="enable"
                        settingValue={config.window.siyuan.enable}
                        type={ItemType.checkbox}
                        on:changed={(e) => {
                            config.window.siyuan.enable = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 打开一个桌面端编辑器 -->
                <Item
                    text={i18n.settings.siyuan.open.desktop.description}
                    title={i18n.settings.siyuan.open.desktop.title}
                >
                    <Input
                        slot="input"
                        settingKey="open-desktop-window"
                        settingValue={i18n.settings.siyuan.open.desktop.text}
                        type={ItemType.button}
                        on:clicked={(e) => plugin.openSiyuanDesktopWindow(e.detail.event)}
                    />
                </Item>

                <!-- 打开一个移动端编辑器 -->
                <Item
                    text={i18n.settings.siyuan.open.mobile.description}
                    title={i18n.settings.siyuan.open.mobile.title}
                >
                    <Input
                        slot="input"
                        settingKey="open-mobile-window"
                        settingValue={i18n.settings.siyuan.open.mobile.text}
                        type={ItemType.button}
                        on:clicked={(e) => plugin.openSiyuanMobileWindow(e.detail.event)}
                    />
                </Item>

                <!-- 是否默认聚焦 -->
                <Item
                    text={i18n.settings.siyuan.focus.description}
                    title={i18n.settings.siyuan.focus.title}
                >
                    <Input
                        slot="input"
                        settingKey="focus"
                        settingValue={config.window.siyuan.focus}
                        type={ItemType.checkbox}
                        on:changed={(e) => {
                            config.window.siyuan.focus = e.detail.value;
                            updated();
                        }}
                    />
                </Item>

                <!-- 默认打开的思源编辑器 -->
                <Item
                    text={i18n.settings.siyuan.editorType.description}
                    title={i18n.settings.siyuan.editorType.title}
                >
                    <Input
                        slot="input"
                        options={[
                            { key: EditorType.mobile, text: i18n.settings.siyuan.editorType.options.mobile },
                            { key: EditorType.desktop, text: i18n.settings.siyuan.editorType.options.desktop },
                            { key: EditorType.window, text: i18n.settings.siyuan.editorType.options.window },
                        ]}
                        settingKey="menuBar"
                        settingValue={config.window.siyuan.editorType}
                        type={ItemType.select}
                        on:changed={(e) => {
                            config.window.siyuan.editorType = e.detail.value;
                            updated();
                        }}
                    />
                </Item>
            </div>
        </Tabs>
    </Panel>
</Panels>

<style lang="less">
</style>
