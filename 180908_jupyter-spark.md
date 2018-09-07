# jupyter-pyspark配置指南

是时候写中文了。
本篇记录的主要是配置jupyter-pyspark的过程中踩的坑。

## 准备条件
首先，我们默认集群已经搭建完毕，且环境变量也配置良好。也就是说，在命令行中输入pyspark就能够启动pyspark-shell.
注意pyspark只支持python2。

## 安装jupyter
由于集群往往搭建在本机，我们直接采用在本机环境搭建jupyter的方法。
既然pyspark已经可以跑了，默认Python2已经安装好

    pip2 install jupyter

## 配置pyspark-jupyter
配置spark环境变量

    export SPARK_HOME=/opt/cloudera/parcels/SPARK2/lib/spark2
    export PATH=$SPARK_HOME/bin:$PATH

    export PYTHONPATH=$SPARK_HOME/python/:$PYTHONPATH
    export PYTHONPATH=$SPARK_HOME/python/lib/py4j-0.8.2.1-src.zip:$PYTHONPATH

SPARK_HOME的地址是你sprak的安装目录，另外在最后一行代码中，py4j-0.8.2.1-src.zip可能会因版本不同而不同，请进入对应地址确认好该文件的名字。 

之后配置选项启动pyspark-jupyter的环境变量

    export PYSPARK_DRIVER_PYTHON="jupyter"
    export PYSPARK_DRIVER_PYTHON_OPTS="notebook --notebook-dir /home/inn/ --allow-root --port 9999"

注意第二个环境变量中，notebook后面的内容实际上是pyspark的设置。--notebook-dir后接的是启动的目录，--port则是pyspark运行的端口。
一般来讲，hadoop所在的账户都没有文件系统权限，所以为保证jupyter创建文件什么的正常，需要改变一下权限。
    
    sudo chomod -R 777 /home/inn
    
之后别忘了开个screen保证jupyter运行
    
    screen -S jupyter-notebook
    
最后启动pyspark。后面的配置选项与常规启动pyspark的时候一致。

    pyspark --executor-memory 10g --total-executor-cores 5 --executor-cores 3 --master yarn
    
不出意外，就能看见jupyter-notebook的提示。

    WARNING: User-defined SPARK_HOME (/opt/cloudera/parcels/SPARK2-2.3.0.cloudera3-1.cdh5.13.3.p0.458809/lib/spark2) overrides detected (/opt/cloudera/parcels/SPARK2/lib/spark2).
    WARNING: Running pyspark from user-defined location.
    [I 18:52:01.931 NotebookApp] JupyterLab extension loaded from /usr/lib/python2.7/site-packages/jupyterlab
    [I 18:52:01.931 NotebookApp] JupyterLab application directory is /usr/share/jupyter/lab
    [I 18:52:01.939 NotebookApp] Serving notebooks from local directory: /home/inn/TST-subway
    [I 18:52:01.939 NotebookApp] The Jupyter Notebook is running at:
    [I 18:52:01.939 NotebookApp] http://localhost:9999/?token=86e832f0ce32f85d701d3b9bceff2d63cc43ac2b66f393be
    [I 18:52:01.939 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
    [W 18:52:01.939 NotebookApp] No web browser found: could not locate runnable browser.
    [C 18:52:01.940 NotebookApp] 
    
    Copy/paste this URL into your browser when you connect for the first time,
    to login with a token:
        http://localhost:9999/?token=86e832f0ce32f85d701d3b9bceff2d63cc43ac2b66f393be

之后，连接服务器并配置端口转发，将9999端口转发至本地，即可通过localhost:9999访问jupyter。

