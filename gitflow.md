# 修改了 dynanfe 里的文件
cd dynanfe
git add .
git commit -m "描述改了什么"
git push

# 修改了 openpi 里的文件
cd dynanfe/openpi
git add .
git commit -m "描述改了什么"
git push myfork main

# 然后回主仓库更新 submodule 引用
cd ..
git add openpi
git commit -m "update openpi submodule"
git push