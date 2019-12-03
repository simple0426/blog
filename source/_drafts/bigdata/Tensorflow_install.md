# 软件下载
* [Anaconda2(python2.7)](https://repo.continuum.io/archive/Anaconda2-5.0.1-Linux-x86_64.sh)

# 安装python
`sudo apt-get install python-pip python-dev`
# 安装anaconda
```
# 一路yes或enter
bash Anaconda2-5.0.0.1-Linux-x86_64.sh
source ~/.bashrc
```
# 安装Theano 
* 安装依赖包：`sudo apt-get install gfortran liblapack-dev`
* 使用conda更新scipy：`conda update scipy`
* 安装Theano：`pip install -i https://pypi.pubyun.com/simple Theano`
* 测试【时间非常长】

```
pip install nose-parameterized
python -c "import theano; theano.test()"
```

# 安装Tensorflow
* 使用conda更新numpy：`conda update numpy`
* 安装tensorflow：`pip install -i https://pypi.pubyun.com/simple tensorflow`

# 安装Keras
* 安装keras：`pip install -i https://pypi.pubyun.com/simple keras`

# 测试
* 验证安装：`import keras`，输出：`Using TensorFlow backend`
* 变更后端计算框架：~/.keras/keras.json

```
"backend": "tensorflow"
"backend": "theano"
```

# 安装参考
* [Ubuntu14.04+Keras+Theano+Tensorflow配置](http://blog.csdn.net/m984789463/article/details/77074451)
