# c-drive-cleanup

> 来源分类：**待确认** ｜ 导出批次：review

Windows C盘空间清理与维护。当用户说"清理C盘/C盘满了/释放C盘空间/C盘空间不足/磁盘清理/清理垃圾"时使用。自动诊断C盘占用、执行系统垃圾清理（临时文件/更新缓存/日志/回收站等19项）、迁移大文件到D盘、清理AI模型缓存、修改浏览器下载路径、DISM组件深度清理，全程保留系统文件与个人资料，输出清理报告。

## 安装

把本文件夹整体复制到 WorkBuddy 技能目录：

```bash
cp -r . ~/.workbuddy/skills/c-drive-cleanup        # 用户级
# 或
cp -r . <项目>/.workbuddy/skills/c-drive-cleanup   # 项目级
```

重启/刷新 WorkBuddy 后即可在对话中触发。

## 说明

- 本技能从本地 WorkBuddy 环境导出，**所有真实密钥已脱敏为占位符**，使用前请配置你自己的 API Key。
- 若来自技能市场（文件夹名以 `__skillhub` 结尾），版权归原作者，请遵守其许可证。
