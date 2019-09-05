---
layout:     post                    
title:      SpecificationBuilder               
subtitle:   简化Spring Data JPA动态查询 
date:       2019-09-05              
author:     KeshawnVan                      
header-img: img/post-bg-keybord.jpg    
catalog: true                       
tags:                               
    - Framework
---

# SpecificationBuilder

## 简介
简化Spring Data JPA动态查询中构建Specification代码比较繁琐的操作。目前已在三家公司五个商业级项目中使用。

## 现有问题
假设有一个domain，里面有如下几个字段:
* Long id
* String name
* Integer age
* String school
* LocalDateTime birthday

如果我们要做一个支持按照标识（精确），年龄（精确），学校（批量），生日（区间），和名字（模糊）进行匹配的动态查询，需要构建一个包含如下字段的condition

* Long id
* String name
* Integer age
* List<String> schools
* LocalDateTime birthdayStart
* LocalDateTime birthdayEnd  
        
之后我们需要编写一段冗长的代码去构建Specification对象

```
Specification specification = new Specification<User>() {
        @Nullable
        @Override
        public Predicate toPredicate(Root root, CriteriaQuery criteriaQuery, CriteriaBuilder criteriaBuilder) {
            List<Predicate> predicates = new ArrayList<>();
            if (userCondition.getId() != null) {
                predicates.add(criteriaBuilder.equal(root.get("id"), userCondition.getId()));
            }
            if (userCondition.getAge() != null) {
                predicates.add(criteriaBuilder.equal(root.get("age"), userCondition.getAge()));
            }
            if (userCondition.getName() != null) {
                predicates.add(criteriaBuilder.like(root.get("name"), "%" + userCondition.getName() + "%"));
            }
            if (userCondition.getBirthdayStart() != null) {
                predicates.add(criteriaBuilder.greaterThanOrEqualTo(root.get("birthday"), userCondition.getBirthdayStart()));
            }
            if (userCondition.getBirthdayEnd() != null) {
                predicates.add(criteriaBuilder.lessThanOrEqualTo(root.get("birthday"), userCondition.getBirthdayEnd()));
            }
            if (CollectionUtils.isNotEmpty(userCondition.getSchools())) {
                predicates.add(root.get("school").in(userCondition.getSchools()));
            }
            return criteriaBuilder.and(predicates.toArray(new Predicate[predicates.size()]));
        }
        };
```      
        
仅仅五个字段的动态SQL，就写26行代码，大多数都是样板代码,实际使用中一个condition往往大于十个字段

## 使用SpecificationBuilder
使用SpecificationBuilder去构建该查询的Specification仅需一行

```
Specification buildSpecification = SpecificationBuilder.buildSpecification(condition);
```

当然你需要在Condition对象中添加注解进行一些必要的描述，condition代码如下

```
  //单数字段做equal操作
  private Long id;
  //使用@Like注解标明进行模糊匹配
  @Like  
  private String name;
  private Integer age;
  //condition中字段名与domain中的字段不一致时使用@Name修正，集合做in操作
  @Name("school") 
  private List<String> schools;
  //@GreaterThanEqual等同于大于等于
  @Name("birthday")
  @GreaterThanEqual 
  private LocalDateTime birthdayStart;
  //@LessThanEqual等同于小于等于
  @Name("birthday")
  @LessThanEqual  
  private LocalDateTime birthdayEnd;
```

## 使用指南
使用注解加在condition的字段上对查询进行必要的说明
* 其中集合类型默认做in操作，单数默认做equal操作
* @Like标明模糊查询，默认在变量左右添加百分号，支持传入参数Like.LEFT,Like.RIGHT进行定制查询
* @GreaterThanEqual标明大于等于，大于使用@GreaterThan
* @LessThanEqual标明小于等于，小于使用@LessThan
* @Page标明分页相关的字段，不参与构建Specification
* @Ignore标明为可忽略字段
* @SmartLike查询参数带有%时使用模糊查询，否则使用精确查询
* @Join指定外键关联
* @NonNull等价于is not null
* @IsNull等价于is null