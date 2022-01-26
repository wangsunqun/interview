## 启动过程
### @SpringBootApplication注解解释
这是一个复合注解：@SpringBootConfiguration、@ComponentScan、@EnableAutoConfiguration  
- @SpringBootConfiguration：继承@Configuration，所以可以在启动类直接定义@Bean 
- @ComponentScan：包扫描，value没填值，默认为使用该注解的类的所在包路径
- @EnableAutoConfiguration：实现自动装配。
  - @AutoConfigurationPackage
    - 就一个作用：创建一个BasePackages类型的Bean，注册到容器。这个类有一个属性，就是启动类所在的包路径。根据作者的注释（原文：用于存储自动配置包以供以后参考的类（例如，通过 JPA 实体扫描器））可以看出这个bean的作用是仅仅存储所在类所在的包路径，供其他组件扫描使用。  
  - @Import(AutoConfigurationImportSelector.class)
    - 这里有个小插曲2.0的低版本跟高版本代码有些许变化，我看2.0.4时候主要逻辑在主类的selectorImport里，2.2.7时候在一个内部类
