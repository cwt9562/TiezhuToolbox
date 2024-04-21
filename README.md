# 铁柱工具箱（沧溟版）



## todo

1. [ ] 将连接步骤移动至底栏，后删除欢迎页
2. [x] 堵速度时区分紫装红装（紫装+9和+12要求12速，红装+9要求10速、+12要求12速）
3. [ ] 自动识别界面位置时，记住上一次成功的界面位置，避免浪费计算
4. [x] 识别时增加loading显示
5. [x] 定时自动循环识别
6. [ ] 添加设置页，实现设置持久化
   1. [ ] 将暴击爆伤的评分标准做成配置项
   2. [ ] 将各级别的准入门槛做成配置项
   3. [ ] 将选中的模拟器配置做成配置项
7. [ ] 将截图操作与OCR操作的代码独立JS文件
8. [ ] 尝试在重构界面算重构后的分数
9. [ ] 尝试识别属性图标
10. [ ] 尝试在背包列表中识别胚子
11. [ ] 尝试在打铁界面识别胚子
   
## 构建

npm install
npm run dev

## 打包

npm run build