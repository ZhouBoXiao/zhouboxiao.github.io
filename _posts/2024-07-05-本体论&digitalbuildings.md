---
layout:     post
title:      "本体论&digitalbuilding使用"
subtitle:   "本体论&digitalbuilding使用"
date:       2024-07-05
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - 洞察
---

# Digital Buildings Project

数字建筑项目是一个开放源代码、Apache许可证的项目，旨在创建一个统一的模式和工具集，用于表示建筑物及其安装设备的结构化信息。目前，数字建筑本体论和工具集的一个版本正在被谷歌用于管理其投资组合中的建筑物。数字建筑项目起源于以可扩展的方式管理大量且异构建筑物组合的需要。该项目旨在实现在建筑物之间轻松移植管理应用程序和分析的目标。这一目标通过结合语义上表达丰富的抽象建模、易于使用的配置语言和健壮的验证工具来实现。数字建筑项目的工作受到了Project Haystack和BrickSchema的启发，并以长期目标保持跨兼容性和/或融合。

在创建“数字建筑”项目时，我们考虑了以下几点： 人可读性、机器可读性与解释、可组合的功能、尺寸分析、正确性验证、跨兼容性

### Project Structure

本项目的结构如下:

- 定义语义数据模型(“术语框”)参数的本体，以及用于构建、验证和将实际设备与特定模型关联的工具。它包含以下格式:
  - YAML格式RDF/OWL格式
- 模型实例配置(又称构建配置文件)，它包含本体和“原始”真实世界数据之间的映射。构建配置文件是“断言框”。
- 支持以下功能的工具:
  - [**ABEL**](https://github.com/google/digitalbuildings/blob/master/tools/abel/README.md):通过从模板化的Google Sheet转换为构建配置文件(并从构建配置文件返回到Google Sheet)，简化了构建配置的构建。
  - [**Explorer**](https://github.com/google/digitalbuildings/blob/master/tools/explorer/README.md)：允许用户浏览本体类型字段，并对不同的本体类型进行比较。
  - [**Instance Validator**](https://github.com/google/digitalbuildings/blob/master/tools/validators/ontology_validator/README.md):使用可选的遥测验证验证DBO的具体应用程序(实例)(即，构建配置文件)。
  - [**Ontology Validator**](https://github.com/google/digitalbuildings/blob/master/tools/validators/ontology_validator/README.md):在更改或扩展时验证本体(目前仅支持YAML格式)。
  - [**RDF/OWL Generator**](https://github.com/google/digitalbuildings/blob/master/tools/rdf_generator/README.md):从YAML本体文件生成RDF版本。
- [Internal Building Representation (IBR)](https://github.com/google/digitalbuildings/blob/master/ibr/README.md)文件格式，用于表示来自不同垂直方向(如空间或资产)的数据。





## 附录

https://github.com/google/digitalbuildings