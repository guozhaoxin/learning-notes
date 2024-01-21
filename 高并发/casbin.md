### PERM

PERM，Policy、Effect、Request、Matchers，是构成 PERM 的4个基础，描述资源和用户之间的关系。



#### Request

Request 定义一个请求参数列表，是一个元组对象，至少包括一个 subject（accessed entity）、object（accessed resource）、action（access method）。

#### Policy

定义访问策略。

#### Matchers

定义请求和 policy 之间的匹配规则。

#### Effect

对于 Matchers 的结果进行一些逻辑运算，得到最终的结果。



### RBAC

Role-Based Access Control， 是一个方法，它根据一个用户拥有的角色，对其能访问的资源进行限制。

#### ACL

Access Control List，用户、行为、资源之间的映射。