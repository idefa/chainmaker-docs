# KubeSimmed Doc

工具使用文档

https://cloud.tencent.com/developer/article/1849548

本地编译：
sphinx-autobuild source build/kubesimmed_doc

多版本编译
sphinx-multiversion source build/html

服务器编译并启动：sphinx-autobuild source ../kubesimmed_doc
服务器仅编译不运行：sphinx-build -b html source ../kubesimmed_doc

curl -sL "https://yihui.org/tinytex/install-bin-unix.sh" | sh
tlmgr option repository https://mirrors.aliyun.com/CTAN/systems/texlive/tlnet

tlmgr install titlesec framed threeparttable wrapfig multirow enumitem bbding titling tabu mdframed tcolorbox textpos import varwidth needspace tocloft ntheorem environ trimspaces collection-fontsrecommended capt-of eqparbox cjk
