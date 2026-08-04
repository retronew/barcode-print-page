# 商品条码打印页面

一个可直接运行的 Vue 3 + TypeScript 页面，支持：

- CODE128、CODE39、EAN-13、EAN-8、UPC、ITF-14、ITF、MSI、Pharmacode
- A4、A5、Letter、常用标签纸和自定义纸张尺寸
- 标签宽高、条码实际宽高、页边距、间距的毫米级配置
- 自动计算最大行列数，或手动指定布局
- 纸张、布局、标签内容和条码格式越界/错误提醒
- 多商品、数量展开、自动分页
- 使用 SVG 条码，打印时按照 CSS `mm` 输出

## 运行

```bash
npm install
npm run dev
```

也可以使用 pnpm：

```bash
pnpm install
pnpm dev
```

## 集成到已有项目

复制 `src/components/BarcodePrintPage.vue`，并安装：

```bash
pnpm add jsbarcode
pnpm add -D @types/jsbarcode
```

## 实际尺寸打印注意事项

页面会动态写入：

```css
@page {
  size: <纸张宽度>mm <纸张高度>mm;
  margin: 0;
}
```

打印对话框中仍需：

1. 缩放选择“实际大小”或 `100%`。
2. 不要选择“适合页面”。
3. 关闭浏览器页眉页脚。
4. 打印机如果存在不可打印边缘，应在页面中设置对应页边距。

首次接入某台打印机时，建议用尺子测量一条 100 mm 测试线，确认驱动没有额外缩放。
