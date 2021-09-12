# Prometheus TSDB (Part 4): Persistent Block and its Index

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-persistent-block-and-its-index/

## 简介

## 持久化 block 是什么 & 在何时创建的


## block 的内容


### 1. `meta.json`

### 2. `chunks`

### 3. `index`

#### A. `TOC`

#### B. `Symbol Table`

#### C. `Series`

#### D. `Label Offset Table` 和 `Label Index i`

##### `Label Index i`


##### `Label Offset Table`

#### E. `Postings Offset Table` 和 `Postings i`

##### `Postings i`

##### `Postings Offset Table`


### 4. `tombstones`

## 尾声


## 代码参考
