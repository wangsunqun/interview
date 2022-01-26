## 启动过程
### @SpringBootApplication注解解释
这是一个复合注解：@SpringBootConfiguration、@ComponentScan、@EnableAutoConfiguration  
- @SpringBootConfiguration：继承@Configuration，所以可以在启动类直接定义@Bean 
- @ComponentScan：包扫描，value没填值，默认为使用该注解的类的所在包路径
- @EnableAutoConfiguration：实现自动装配。
  - @AutoConfigurationPackage
    
  - @Import(AutoConfigurationImportSelector.class)
