# 基于 opencode 的个人 AI 助理

## 环境准备

### feishu cli

```sh
npm install -g @larksuite/cli
npx skills add https://open.feishu.cn -y -a opencode

lark-cli config init
lark-cli auth login --recommend
lark-cli auth status
```
