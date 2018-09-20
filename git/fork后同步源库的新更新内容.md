# 步骤：
- 查看远程状态  
  `git remote -v`
- 给 fork 添加源库的clone地址  
  `git remote add upstream 源库的clone地址`
- 再次查看状态确认是否配置成功
- 从源库获取更新到master分支（pull相当于fetch+merge）  
  `git pull upstream master`