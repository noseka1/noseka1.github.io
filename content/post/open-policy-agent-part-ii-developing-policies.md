+++
title = "Oen Policy Agent, Part II — Developing Policies"
date = "2019-10-27"
slug = "2019/10/27/open-policy-agent-part-ii-developing-policies"
Categories = [ "development" ]
+++

In the [previous part](/blog/2019/10/08/open-policy-agent-part-i-the-introduction/) of the series, we explored Open Policy Agent and implemented an ACL-based access control for our application. In this entry, I am going to share with you some of the discoveries that I made while evaluating Open Policy Agent in regards to policy design and development.

<!--more-->

# Notes on Policy Design

After evaluating policy rules, OPA returns a result of the policy decision to your application. This result is a JSON structure. Based on your requirements, this JSON structure can contain a single member holding a *true* or *false* (authorized/not authorized) value. However, you can create policies whose evaluation results in an arbitrarily complex JSON document. For example, OPA can return a list of nodes on which Kubernetes should schedule a workload.

In microservice applications, OAuth 2.0 is a rather popular authorization framework used to secure service’s APIs. It typically leverages JSON Web Tokens (JWT) to convey claims. OPA comes with built-in functions that can decode the token and validate its signature and expiration time. Furthermore, your policy rules can make decisions based on the claims included in the token. Just forward the token as an input to OPA and offload the entire token processing from your application!

OPA makes policy decisions based on the data stored in memory. In the case of large data sets, replicating all the data in memory can be impractical. While evaluating policy rules, is OPA able to reach out to an external data store to get additional data for decision making? For example, send a query to LDAP to grab additional attributes or look up data in an SQL database? Based on my research, I think there are two possible approaches for leveraging external data sources in OPA. First, there is a built-in [HTTP](https://www.openpolicyagent.org/docs/latest/language-reference/#http) function that can fetch data from external HTTP services during policy evaluation. Second, you can leverage Partial Evaluation as described in this [blog post](https://blog.openpolicyagent.org/write-policy-in-opa-enforce-policy-in-sql-d9d24db93bf4). While partially evaluating policies, OPA doesn’t return a complete policy decision but instead it returns a set of conditions. It is left to you to translate this set of conditions into a query appropriate for your data store and execute the query in order to obtain the final policy decision. Note that regardless of which approach you choose, reaching out to external data stores will have negative impact on latency and reliability of your solution. Caching data in OPA’s memory is always a better option assuming that it suits your use case.

If you have raw data that would be difficult to write a policy against, you can pre-process that data into a form that better suits the policy writing before importing it into OPA. Moreover, if you have multiple sources of data, e.g. data from LDAP and Active Directory, you can merge them outside of OPA and load the merged form into OPA.

RBAC (Role-Based Access Control) and ABAC (Attribute-Based Access Control) are two frequently used policy models. Are you wondering if you can implement them using OPA? Of course you can! Follow these two links to find sample implementations of [RBAC](https://www.openpolicyagent.org/docs/latest/comparison-to-other-systems/#role-based-access-control-rbac) and [ABAC](https://www.openpolicyagent.org/docs/latest/comparison-to-other-systems/#attribute-based-access-control-abac).

Hierarchical group permissions are commonly found in practice, e.g. parent group permissions are a superset of child group permissions. These models can be elegantly described using recursive rules. However, at the time of this writing, OPA doesn’t support [recursion in policies](https://github.com/open-policy-agent/opa/issues/947).

# Developing policies

While learning the OPA’s Rego language, I appreciated the built-in interactive shell (REPL) that I could use to write and test my policies instantly. Just type `opa run` and you are good to go. Alternatively, you can go on-line and utilize the [Rego Playground](https://play.openpolicyagent.org/), too.

If you are dealing with complex policies, how do you ensure that you implemented your policies correctly? OPA [allows](https://www.openpolicyagent.org/docs/latest/how-do-i-test-policies/) you to write test cases which you can run against your policies. You can use data mocking and calculate test coverage. See also the command `opa test`.

Is the evaluation of your policies too slow? OPA comes with a [profiler](https://www.openpolicyagent.org/docs/latest/how-do-i-test-policies/#profiling) to report on time spent on evaluating policy expressions. See also the `opa eval` command.

OPA comes with a formatting tool `opa fmt` to format Rego policy files. You don’t need to fight battles with other developers about how the Rego files should be formatted!

OPA is a relatively new project, however, additional tooling and integrations with OPA are showing up quickly. If you like to use Visual Studio Code, there is a feature-rich [VS Code plugin](https://marketplace.visualstudio.com/items?itemName=tsandall.opa) available for you. Rego syntax highlighting is available for several other editors like VIM, [Atom](https://github.com/open-policy-agent/opa/tree/master/misc/syntax/atom), and [TextMate](https://github.com/open-policy-agent/opa/tree/master/misc/syntax/textmate).

# Conclusion

In this blog post, I shared with you several tips and approaches for how to design policies in Open Policy Agent. In the [final article](/blog/2019/12/03/open-policy-agent-part-iii-integrating-with-your-application/) in the series we will focus on how you can integrate Open Policy Agent with your application.

If you have any comments or questions, please use the comment section below. I look forward to hearing from you.
