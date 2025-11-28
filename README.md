# AIP Project整体介绍

# 1. 项目介绍
本项目作为中国政府国家重点研发计划的智能体生态的一部分，由北京邮电大学牵头，依托中国电子技术标准化研究院开发，于2025年发布第一版，版本号为v1.0。

文档编写单位：北京邮电大学人工智能学院，中国电子技术标准化研究院

文档编写者：刘军（北京邮电大学），高歌（中国电子技术标准化研究院），李珂（北京邮电大学），陈科良（北京邮电大学），禹可（北京邮电大学）。
# 2. 协议概述
智能体互联是智能体通过标准化的协议或接口，与其他智能体连接共同完成指定任务的过程。目的是为了突破单智能体能力限制，同时打破厂商自有多智能体框架的束缚，以开放互联的形式构建一个智能体之间可以实现平等互通、互联协作、互惠互利的平台。未来，智能体之间可以自组织、自协商形成一种高效协作的网络，实现根据用户任务需求动态调整资源分配和协作方式，提高智能体系统的整体能力和效率。

本项目从未来智能体互联将成为关键性网络基础设施的愿景出发，尝试从更加全局化的视角，提出并设计了一套面向智能体互联的智能体互联协议族（Agent Interconnection Protocol），涵盖智能体身份码规范、智能体能力描述规范、智能体可信注册规范、智能体身份认证规范、智能体发现协议规范、智能体交互协议规范、发现与注册数据同步规范。

本项目还开源了配套参考实现，具体包括智能体注册服务端(ACPs-Registry-Server)、智能体身份认证客户端（ACPs-CA-Client）、智能体身份认证服务端（ACPs-CA-Server）、智能体发现服务端(ACPs-Discovery-Server)、智能体示范应用（ACPs-Demo-Project）等，以弥补现有智能体协议的不足。为智能体互联的坚实发展提供新的思路和方法。我们欢迎智能体技术爱好者广泛交流探讨，共同完善该协议族。

# 3. 智能体协作协议体系定义的规范

智能体协作协议体系（Agent Collaboration Protocols，ACPs）是为保障异构智能体之间高效协作、支持多样化智能体互联应用而设计的标准化通信与交互协议体系与上图中实体属性和实体间接口序号对应，ACPs 中定义了如下规范和协议：

(1)[智能体协作协议体系（Agent Collaboration Protocols，ACPs）](01/ACPs-spec.md)

(2)[智能体身份码（Agent Identity Code，AIC）规范](02/ACPs-spec-AIC.md)

(3)[智能体能力描述（Agent Capability Specification，ACS）规范](03/ACPs-spec-ACS.md)

(4)[智能体可信注册（Agent Trusted Registration，ATR）规范](04/ACPs-spec-ATR.md)

(5)[智能体身份认证（Agent Identity Authentication，AIA）规范](05/ACPs-spec-AIA.md)

(6)[智能体发现协议（Agent Discovery Protocol，ADP）规范](06/ACPs-spec-ADP.md)

(7)[智能体交互协议（Agent Interaction Protocol，AIP）规范](07/ACPs-spec-AIP.md)

(8)[发现与注册数据同步（Discovery and Registration Coordination，DRC）规范](08/ACPs-spec-DRC.md)

# 4. demo使用
为了您快速体验本协议族，我们提供了一个demo，其代码仓库位于 [ACPs-Demo-Project](https://github.com/aip-pub/ACPs-Demo-Project),在这里我们提供了能快速体验demo的流程。

# 5. 补充说明

本项目介绍了北京邮电大学人工智能学院智能体互联研究小组(刘军、李珂、陈科良、禹可、胡晓峰、马镝)提出并初步定义的面向海量智能体互联协作的协议族是从未来智能体互联能成为关键性网络基础设施的角度出发，尝试从更加全局化的视野，为智能体互联的坚实发展提供新的思路和方法。需要特别说明的是，本项目是研究团队首次公布基本框架和思路，在具体实现层面可能还存在一定的不足，我们欢迎广大技术爱好者指正和交流，共同完善该协议族。

# 附件

[智能体互联核心概念](Guide/CoreConcepts.md)   
