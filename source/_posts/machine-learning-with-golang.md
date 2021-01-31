---
title: "实践: 使用Golang进行机器学习"
date: 2021-01-31 11:16:03
---

主流当下进行机器学习当然都是用的Python, 我的大部分工作也基本都是使用Python完成的. 但是Python的性能实在是太捉襟见肘, 所以我后面转向了Golang. 使用Golang进行机器学习对我而言是能更快地学习两者. 接下来我就来介绍在这过程中的学习和记录.

## 开发环境配置

我的机器配置是双AMD的3700x+5700xt, 安装的黑苹果macOS Catalina 10.15.7这其实还挺麻烦的. 

* Tensorflow: 只是需要注意Python的版本. Tensorflow的支持版本是3.5-3.8我的版本是3.9, 所以我还需要安装3.8版本的Python

  ```bash
  # brew install python@3.8
  # echo 'export PATH="/usr/local/opt/python@3.8/bin:$PATH"' >> ~/.zshrc
  pip install --upgrade pip
  pip install tensorflow
  ```

* Mercurial: `brew install mercurial`

* Minikube: `brew install minikube`

* Pachyderm: `brew tap pachyderm/tap && brew install pachyderm/tap/pachctl@1.9`

## Golang进行数据处理

如果是手动解析数据的话, 需要手动定义数据格式, 并且对于异常数据的错误处理会很麻烦. 

* csv文件使用`go-gota/gota/dataframe`进行处理. 

``` go
irisDF := dataframe.ReadCSV(f)
filter := dataframe.F{
	Colname:    "species",
	Comparator: "==",
	Comparando: "Iris-versicolor",
}
versicolorDF := irisDF.Filter(filter).Select([]string{"sepal_width", "species"}).Subset([]int{0, 1, 2})
```

* json文件使用`tidwall/gjson`和`tidwall/sjson`可以不需要定义schema就可以读取/设置

``` go
json, err := ioutil.ReadFile("../data/station_status.json")
value := gjson.Get(string(json), "last_updated")

modified_json, _ := sjson.Set(string(json), "last_updated", "123456")
modified_value := gjson.Get(modified_json, "last_updated")
```

* 数据缓存: 使用`patrickmn/go-cache`在内存中缓存一系列数值

``` go
c := cache.New(5*time.Minute, 30*time.Second)
c.Set("mykey", "myvalue", cache.DefaultExpiration)
v, found := c.Get("mykey")
```

* 数据本地缓存: `boltdb/bolt`被归档了, 所以我正在考虑其替代品`Badger`

* 数据版本控制: `minikube start && pachctl deploy local` 在本地部署一个单机节点