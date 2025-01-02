开发阶段,windows PC=================
1.生成页面
make html

2.启动服务
sphinx-autobuild source build/html --open-browser

3. 访问页面


提交代码流程==================
git add .
git commit -m "demo"

git push -u origin main


重新构建==================
托管服务构建即可

ssh -T   git@github.com
$ ssh -vT   git@github.com
ssh -T -p 443 git@ssh.github.com