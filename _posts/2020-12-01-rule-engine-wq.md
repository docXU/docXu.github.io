---
layout:     post
title:      "如何利用wq规则引擎实现业务决策"
subtitle:   " \"将核心业务逻辑的控制从后端搬到前端\""
date:       2020-12-01 11:37:00
author:     "MattX"
header-img: "img/rule-engine-wq/bg.png"
catalog:    true
tags:
    - 设计模式
    - 规则引擎
    - 重构
---

1. 引入maven依赖
    ```Java
        <dependency>
            <groupId>com.xxw.phoenix</groupId>
            <artifactId>rule-engine-service-springboot</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
    ```
2. 添加自动导入注解RuleEngineAutoConfiguration.class以及yml配置
    ```Java
        @Configuration
        @Import(RuleEngineAutoConfiguration.class)
        public class RuleEngineConfig {
            @Bean
            public FactsLifeEventHandler factsLifeEventHandler(){
                return new FactsLifeEventHandler();
            }
        }
    ```
    ```Yml
        phx:
            rule-engine:
                # 保证各节点的规则和模型最终一致
                update-rules-job-enable: true 
                # 配置规则引擎模式使用的线程池参数
                thread-pool:    
                    bypassWorkQueueCapacity: 1024
                    defaultBuildFactsCorePoolSize: 32
                    defaultBuildFactsMaxPoolSize: 64
                    defaultBuildFactsQueueCapacity: 128
                    defaultBuildFactsTimeoutMillis: 100
                # redis工具类参数设置，优先级低，如果系统已有redisTemplate, 则不会自建一个
                redis:
                    cacheKeyPrefix: riskgo.
                    access-monitor-timeout: 100
                    maxTotal: 20
                    maxWait: 5000
                    maxIdle: 2
                    minIdle: 2
                    softMinEvictableIdleTimeMillis: 5000
                    soTimeoutMs: 1000
                    host: 172.30.1.212
                    port: 6379
                    password: xxw
                    testOnCreate: false
                    testOnBorrow: false
                    testOnReturn: false
                    testWhileIdle: false
    ```
3. 扫描规则引擎的Jpa方法和实体
    * 规则引擎的默认springboot实现依赖mysql保存规则和模型对象，需要管理这些实体对象，自此，规则引擎的接入完成，接下来就可以编写规则以及获取模型实现业务方的决策逻辑了。
    ```Java
        @EnableJpaRepositories(
        entityManagerFactoryRef = "entityManagerFactoryMysql",
        transactionManagerRef = "transactionManagerMysql",
        basePackages = {
            "com.xxw.rcs.riskgo.domain.*",
            "com.xxw.phoenix.rule.engine.*"  // 添加扫描此路径下的repository
        })
        ...
        @Primary
        @Bean(name = "entityManagerFactoryMysql")
        public LocalContainerEntityManagerFactoryBean 
        entityManagerFactoryMysql(EntityManagerFactoryBuilder builder) {
            return builder
                .dataSource(mysqlDataSource())
                .packages(
                    "com.xxw.rcs.riskgo.domain",
                    "com.xxw.phoenix.rule.engine") // 添加扫描此路径下的entity
                .persistenceUnit("riskgoPersistenceUnit")
                .properties(Collections.singletonMap("hibernate.hbm2ddl.auto", "update"))
                .build();
        }
    ```
4. 定义模型和规则
    * 根据业务场景需要创建meta模型@FactMeta，通过bizType = RiskgoRuleBizScene.riskTxnReceive指定业务场景（业务场景是用于获取场景下的规则以及模型的原始数据）
        ```Java
            /**
            * riskgo推送的风险交易记录
            *
            * @author xuxiongwei
            */
            @Data
            @Accessors(chain = true)
            @FactMeta(bizType = RiskgoRuleBizScene.riskTxnReceive)
            public class RiskTxnMeta {
                /**
                * gmtOccur 风险识别时间(long)
                * complainText 投诉描述 （只有在投诉riskType下才有
                * complainTime 投诉时间
                */
                private Date gmtOccur;

                private String complainText;

                private String complainTime;

                private String tradeNos;

                private String sourceId;

                private String externalId;

                private String pid;

                private String sMid;

                private String riskType;

                private String riskLevel;
            }
        ```
    * 定义业务模型@FactModel， 需要在业务规则中进行关系运算决策的对象
        ```Java
            /**
            * 统计商户的风险交易发生情况
            *
            * @author xuxiongwei
            */
            @Data
            @ToString(callSuper = true)
            @FactModel(isDynamicFact = true)
            public class AliRiskSummary extends DynamicFactQuery {
                /**
                * 查询结果
                */
                private Integer cnt;

                private Integer tradeNoCnt;

                /**
                * *
                * * 以下统计条件
                * *
                **/

                private List<String> riskTypeIn;

                private List<String> riskLevelIn;

                private String timeRange;

                private String startTime;

                private String endTime;
            }
        ```
5. 编写模型查询参数校验和获取逻辑
    * 静态模型（可以通过meta信息直接获取的模型）
    * 在FactsBuildService的实现Bean中，定义一个注解了@FactBuilder的方法，方法参数是业务场景模型对象RiskTxnMeta，返回对象是目标模型MchtInfo
        ```Java
        @Component
        @Slf4j
        public class MchtInfoBuilder implements FactsBuildService {
            @FactBuilder
            public MchtInfo buildMcht(RiskTxnMeta riskTxnMeta) {
                return new MchtInfoMcht(riskTxnMeta);
            }
        }
        ```
    * 动态模型（需要通过meta信息和表达式填写的查询参数动态获取的模型）
    * 在继承了AbstractReasonableBuilder<T>的实现Bean中，编写验证逻辑validateDynamic以及一个注解了@FactBuilder的方法，方法参数
        ```Java
            @Component
            @Slf4j
            public class AliRiskSummaryBuilder 
                extends AbstractReasonableBuilder<AliRiskSummary> {

                @Autowired
                private AliProcessRecordService aliProcessRecordService;

                @Override
                public void validateDynamic(Object o) throws RuntimeException {
                    AliRiskSummary riskSummary = (AliRiskSummary) o;
                    ParserUtils.validateTimeRange(riskSummary.getTimeRange(),
                        riskSummary.getStartTime(),
                        riskSummary.getEndTime());

                    if ((riskSummary.getRiskLevelIn() == null 
                            || riskSummary.getRiskLevelIn().isEmpty())
                        && (riskSummary.getRiskTypeIn() == null 
                            || riskSummary.getRiskTypeIn().isEmpty())) {
                        throw new RuntimeException("需要指定RiskLevelIn或者RiskTypeIn的值");
                    }
                }

                @FactBuilder
                public AliRiskSummary riskSummary(RiskTxnMeta riskTxnMeta,
                    DynamicMVELRule dynamicMVELRule,
                    AliRiskSummary summary) {
                        // summary就是通过了表达式初始化以及validateDynamic校验的动态模型对象
                        // 编写对象获取逻辑... 这里演示，直接省略查询逻辑
                        summary.setTradeNoCnt(getByMetaSmid(riskTxnMeta.getSmid()));
                        summary.setCnt((int) dbCnt);
                        return summary;
                }
        ```
    6. 编写规则
        ![规则配置](/img/rule-engine-wq/config.png)
    7. 注入RuleEngineService进行决策
        ```Java
            EngineCheckResponse response = ruleEngineService.check(RiskgoRuleBizScene.riskTxnReceive, riskTxnMeta);
            // AUTO_BLOCK_MCHT 对应了执行选项的拉黑商户
            // 规则若命中了XW01则会在ActionsSet中添加执行选项AUTO_BLOCK_MCHT
            if (response.getActionSet().contains(RiskgoRuleBizAction.AUTO_BLOCK_MCHT)) {
                try {
                    // 拉黑商户
                    shareLockableMcht(aliProcessRecord, configService.getAutoResponseProccessCode());
                } catch (Exception e) {
                    log.error("Share lockable mcht-list error.", e);
                }
            }
        ```