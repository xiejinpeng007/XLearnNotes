>Swagger 是一个统一前后端用于生成文档和代码的工具，它使用 yaml / json 作为描述语言 通过 [OpenAPI Specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#exampleObject) 来描述 API，最后使用 Codegen 根据不同的配置来生成各种 language、library 的 Code、Docs.

##### 其最理想的情况则是只需一份描述文件(yaml/json) 生成 后端、前端（android ios web...）的代码和文档，这样的话保证了前后端的统一，且需要升级改动也只需要修改 yaml 文件。

### YAML
JSON 都已经很熟悉了，虽然 Swagger 可以使用 JSON 作为描述语言，但是因为 YAML 更为简洁直观，所以更推荐 YAML。
YAML 的基本语法并不复杂，这里介绍一些基本语法：  

* yaml 文件 以`---`开始 `...`结尾
* 同一级别的成员（如 list 成员）可以通过`"- " `来辨识
* 注释以`#`开头的一行

* list:

```
fruits:
    - Apple
    - Orange
    - Strawberry
    - Mango
```
* key / value

```
martin:
    name: Martin D'vloper
    job: Developer
    skill: Elite
```

* list / map 混合使用

```
-  martin:
    name: Martin D'vloper
    job: Developer
    skills:
      - python
      - perl
      - pascal
-  tabitha:
    name: Tabitha Bitumen
    job: Developer
    skills:
      - lisp
      - fortran
      - erlang
```

* list / map 的简写

```
martin: {name: Martin D'vloper, job: Developer, skill: Elite}
fruits: ['Apple', 'Orange', 'Strawberry', 'Mango']
```

* boolean 值的写法没有严格限制

```
create_key: yes
needs_agent: no
knows_oop: True
likes_emacs: TRUE
uses_cvs: false
```

* `|`使用换行 `>`忽略换行

```
include_newlines: |
            exactly as you see
            will appear these three
            lines of poetry

ignore_newlines: >
            this is really a
            single line of text
            despite appearances
```

### OpenAPI-Specification

> OpenAPI 是一套用于描述 RESTful APIs 的规范。  
> 
> The OpenAPI Specification (OAS) defines a standard, language-agnostic interface to RESTful APIs which allows both humans and computers to discover and understand the capabilities of the service without access to source code, documentation, or through network traffic inspection. When properly defined, a consumer can understand and interact with the remote service with a minimal amount of implementation logic.
>
>An OpenAPI definition can then be used by documentation generation tools to display the API, code generation tools to generate servers and clients in various programming languages, testing tools, and many other use cases.

只要符合 OpenAPI 规范的都可以生成各类代码、文档、工具。
目前 OpenAPI 最新 3.0.0 ,对比 2.0 对 API 结构进行了调整。  
文档内容繁多请参考 [OpenAPI-Specification 3.0.0](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md)

下面是 Uber API 的 example

```
# this is an example of the Uber API
# as a demonstration of an API spec in YAML
openapi: "3.0.0"
info:
  title: Uber API
  description: Move your app forward with the Uber API
  version: "1.0.0"
servers:
  - url: https://api.uber.com/v1
paths:
  /products:
    get:
      summary: Product Types
      description: The Products endpoint returns information about the Uber products offered at a given location. The response includes the display name and other details about each product, and lists the products in the proper display order.
      parameters:
        - name: latitude
          in: query
          description: Latitude component of location.
          required: true
          schema:
            type: number
            format: double
        - name: longitude
          in: query
          description: Longitude component of location.
          required: true
          schema:
            type: number
            format: double
      security: 
        - apikey: []
      tags: 
        - Products
      responses:  
        '200':
          description: An array of products
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ProductList"
        default:
          description: Unexpected error
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
  /estimates/price:
    get:
      summary: Price Estimates
      description: The Price Estimates endpoint returns an estimated price range for each product offered at a given location. The price estimate is provided as a formatted string with the full price range and the localized currency symbol.<br><br>The response also includes low and high estimates, and the [ISO 4217](http://en.wikipedia.org/wiki/ISO_4217) currency code for situations requiring currency conversion. When surge is active for a particular product, its surge_multiplier will be greater than 1, but the price estimate already factors in this multiplier.
      parameters:
        - name: start_latitude
          in: query
          description: Latitude component of start location.
          required: true
          schema:
            type: number
            format: double
        - name: start_longitude
          in: query
          description: Longitude component of start location.
          required: true
          schema:
            type: number
            format: double
        - name: end_latitude
          in: query
          description: Latitude component of end location.
          required: true
          schema:
            type: number
            format: double
        - name: end_longitude
          in: query
          description: Longitude component of end location.
          required: true
          schema:
            type: number
            format: double
      tags: 
        - Estimates
      responses:  
        '200':
          description: An array of price estimates by product
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: "#/components/schemas/PriceEstimate"
        default:
          description: Unexpected error
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
  /estimates/time:
    get:
      summary: Time Estimates
      description: The Time Estimates endpoint returns ETAs for all products offered at a given location, with the responses expressed as integers in seconds. We recommend that this endpoint be called every minute to provide the most accurate, up-to-date ETAs.
      parameters:
        - name: start_latitude
          in: query
          description: Latitude component of start location.
          required: true
          schema:
            type: number
            format: double
        - name: start_longitude
          in: query
          description: Longitude component of start location.
          required: true
          schema:
            type: number
            format: double
        - name: customer_uuid
          in: query
          schema:
            type: string
            format: uuid
          description: Unique customer identifier to be used for experience customization.
        - name: product_id
          in: query
          schema:
            type: string
          description: Unique identifier representing a specific product for a given latitude & longitude.
      tags: 
        - Estimates
      responses:  
        '200':
          description: An array of products
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: "#/components/schemas/Product"
        default:
          description: Unexpected error
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
  /me:
    get:
      summary: User Profile
      description: The User Profile endpoint returns information about the Uber user that has authorized with the application.
      tags: 
        - User
      responses:
        '200':
          description: Profile information for a user
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Profile"
        default:
          description: Unexpected error
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
  /history:
    get:
      summary: User Activity
      description: The User Activity endpoint returns data about a user's lifetime activity with Uber. The response will include pickup locations and times, dropoff locations and times, the distance of past requests, and information about which products were requested.<br><br>The history array in the response will have a maximum length based on the limit parameter. The response value count may exceed limit, therefore subsequent API requests may be necessary.
      parameters:
        - name: offset
          in: query
          schema:
            type: integer
            format: int32
          description: Offset the list of returned results by this amount. Default is zero.
        - name: limit
          in: query
          schema:
            type: integer
            format: int32 
          description: Number of items to retrieve. Default is 5, maximum is 100.
      tags: 
        - User
      responses:
        '200':
          description: History information for the given user
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Activities"
        default:
          description: Unexpected error
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
components:
  securitySchemes:
    apikey:
      type: apiKey
      name: server_token
      in: query
  schemas:
    Product:
      properties:
        product_id:
          type: string
          description: Unique identifier representing a specific product for a given latitude & longitude. For example, uberX in San Francisco will have a different product_id than uberX in Los Angeles.
        description:
          type: string
          description: Description of product.
        display_name:
          type: string
          description: Display name of product.
        capacity:
          type: integer
          description: Capacity of product. For example, 4 people.
        image:
          type: string
          description: Image URL representing the product.
    ProductList:
      properties:
        products:
          description: Contains the list of products
          type: array
          items: 
            $ref: "#/components/schemas/Product"
    PriceEstimate:
      properties:
        product_id:
          type: string
          description: Unique identifier representing a specific product for a given latitude & longitude. For example, uberX in San Francisco will have a different product_id than uberX in Los Angeles
        currency_code:
          type: string
          description: "[ISO 4217](http://en.wikipedia.org/wiki/ISO_4217) currency code."
        display_name:
          type: string
          description: Display name of product.
        estimate: 
          type: string
          description: Formatted string of estimate in local currency of the start location. Estimate could be a range, a single number (flat rate) or "Metered" for TAXI.
        low_estimate:
          type: number
          description: Lower bound of the estimated price.
        high_estimate:
          type: number
          description: Upper bound of the estimated price.
        surge_multiplier:
          type: number
          description: Expected surge multiplier. Surge is active if surge_multiplier is greater than 1. Price estimate already factors in the surge multiplier.
    Profile:
      properties:
        first_name:
          type: string
          description: First name of the Uber user.
        last_name:
          type: string
          description: Last name of the Uber user.
        email:
          type: string
          description: Email address of the Uber user
        picture:
          type: string
          description: Image URL of the Uber user.
        promo_code:
          type: string
          description: Promo code of the Uber user.   
    Activity:
      properties:
        uuid:
          type: string
          description: Unique identifier for the activity
    Activities:
      properties:
        offset:
          type: integer
          format: int32
          description: Position in pagination.
        limit:
          type: integer
          format: int32
          description: Number of items to retrieve (100 max).
        count:
          type: integer
          format: int32
          description: Total number of items available.
        history:
          type: array
          items:
            $ref: "#/components/schemas/Activity"
    Error:
      properties:
        code:
          type: string
        message:
          type: string
        fields:
          type: string
```


这里将 OpenAPI 3.0 规范中把重点使用的 api 整理了一张思维导图（还不完善，会持续更新）

![OpenAPI 3.0.0.png](http://upload-images.jianshu.io/upload_images/1447881-03d1619fc6f6f670.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Swagger 

Swagger 实际上包含了一系列的工具 Editor Codegen UI ...

* Editor 用于使用 OpenAPI 编辑 yaml 
* Codegen 用于生成不同的 language library 的代码
* UI 用于生成文档

下面介绍一下 Codegen 的使用：

1. 首先下载 [swagger-codegen-cli.jar](https://oss.sonatype.org/content/repositories/snapshots/io/swagger/swagger-codegen-cli/3.0.0-SNAPSHOT/)
2. 确保装好了 maven
3. 准备 swagger.yaml
4. 编写 config.json 配置文件  
   因为之前提到 Swagger 可以生成各种 lang lib 的代码，所以这里便是进行此类配置:  
   查看配置 help  
   
   ```
   java -jar swagger-codegen-cli.jar config-help -l java
   ```
   
   ```
   CONFIG OPTIONS
        sortParamsByRequiredFlag
            Sort method arguments to place required parameters before optional parameters. (Default: true)
        ensureUniqueParams
            Whether to ensure parameter names are unique in an operation (rename parameters that are not). (Default: true)
        allowUnicodeIdentifiers
            boolean, toggles whether unicode identifiers are allowed in names or not, default is false (Default: false)
        modelPackage
            package for generated models
        apiPackage
            package for generated api classes
        invokerPackage
            root package for generated code
        groupId
            groupId in generated pom.xml
        artifactId
            artifactId in generated pom.xml
        artifactVersion
            artifact version in generated pom.xml
        artifactUrl
            artifact URL in generated pom.xml
        artifactDescription
            artifact description in generated pom.xml
        scmConnection
            SCM connection in generated pom.xml
        scmDeveloperConnection
            SCM developer connection in generated pom.xml
        scmUrl
            SCM URL in generated pom.xml
        developerName
            developer name in generated pom.xml
        developerEmail
            developer email in generated pom.xml
        developerOrganization
            developer organization in generated pom.xml
        developerOrganizationUrl
            developer organization URL in generated pom.xml
        licenseName
            The name of the license
        licenseUrl
            The URL of the license
        sourceFolder
            source folder for generated code
        localVariablePrefix
            prefix for generated code members and local variables
        serializableModel
            boolean - toggle "implements Serializable" for generated models (Default: false)
        bigDecimalAsString
            Treat BigDecimal values as Strings to avoid precision loss. (Default: false)
        fullJavaUtil
            whether to use fully qualified name for classes under java.util. This option only works for Java API client (Default: false)
        hideGenerationTimestamp
            hides the timestamp when files were generated
        withXml
            whether to include support for application/xml content type. This option only works for Java API client (resttemplate) (Default: false)
        dateLibrary
            Option. Date library to use
                joda - Joda (for legacy app only)
                legacy - Legacy java.util.Date (if you really have a good reason not to use threetenbp
                java8-localdatetime - Java 8 using LocalDateTime (for legacy app only)
                java8 - Java 8 native JSR310 (preferred for jdk 1.8+) - note: this also sets "java8" to true
                threetenbp - Backport of JSR310 (preferred for jdk < 1.8)
        java8
            Option. Use Java8 classes instead of third party equivalents
                true - Use Java 8 classes such as Base64
                false - Various third party libraries as needed
        useRxJava
            Whether to use the RxJava adapter with the retrofit2 library. (Default: false)
        useRxJava2
            Whether to use the RxJava2 adapter with the retrofit2 library. (Default: false)
        parcelableModel
            Whether to generate models for Android that implement Parcelable with the okhttp-gson library. (Default: false)
        usePlay24WS
            Use Play! 2.4 Async HTTP client (Play WS API) (Default: false)
        supportJava6
            Whether to support Java6 with the Jersey1 library. (Default: false)
        useBeanValidation
            Use BeanValidation API annotations (Default: false)
        performBeanValidation
            Perform BeanValidation (Default: false)
        useGzipFeature
            Send gzip-encoded requests (Default: false)
        useRuntimeException
            Use RuntimeException instead of Exception (Default: false)
        library
            library template (sub-template) to use (Default: okhttp-gson)
                jersey1 - HTTP client: Jersey client 1.19.4. JSON processing: Jackson 2.8.9. Enable Java6 support using '-DsupportJava6=true'. Enable gzip request encoding using '-DuseGzipFeature=true'.
                feign - HTTP client: OpenFeign 9.4.0. JSON processing: Jackson 2.8.9
                jersey2 - HTTP client: Jersey client 2.25.1. JSON processing: Jackson 2.8.9
                okhttp-gson - HTTP client: OkHttp 2.7.5. JSON processing: Gson 2.8.1. Enable Parcelable models on Android using '-DparcelableModel=true'. Enable gzip request encoding using '-DuseGzipFeature=true'.
                retrofit - HTTP client: OkHttp 2.7.5. JSON processing: Gson 2.3.1 (Retrofit 1.9.0). IMPORTANT NOTE: retrofit1.x is no longer actively maintained so please upgrade to 'retrofit2' instead.
                retrofit2 - HTTP client: OkHttp 3.8.0. JSON processing: Gson 2.6.1 (Retrofit 2.3.0). Enable the RxJava adapter using '-DuseRxJava[2]=true'. (RxJava 1.x or 2.x)
                resttemplate - HTTP client: Spring RestTemplate 4.3.9-RELEASE. JSON processing: Jackson 2.8.9
                resteasy - HTTP client: Resteasy client 3.1.3.Final. JSON processing: Jackson 2.8.9

   ```
  
   几个主要的配置参数：
   
    * library，生成的代码支付的类，有jersey1、jersey2、okhttp-gson、resttemplate、resteasy、feign、retrofit、retrofit2等几种类型，我们选择的retrofit2
    * developerName，开发者名字，会出现在代码文件里
    * developerEmail，开发者邮箱，会出现在代码文件里
    * developrOrganization，开发者组织，会出现在代码里
    * invokerPackage，项目的包名
    * apiPackage，生成的***Api.java文件的包名
    * modelPackage，生成的数据模型java文件包名
    * dateLibrary，时间使用的类开
    * useRxJava，是否使用rxjava生成api接口
    * useRxJava2，是否使用rxjava2的方式调用接口

5. generate 生成代码
   首先打印参数信息
   
   ```
   java -jar swagger-codegen-cli.jar generate help
   ```
   
   ```
   NAME
        swagger-codegen-cli generate - Generate code with chosen lang
    SYNOPSIS
        swagger-codegen-cli generate
                [(-a <authorization> | --auth <authorization>)]
                [--additional-properties <additional properties>...]
                [--api-package <api package>] [--artifact-id <artifact id>]
                [--artifact-version <artifact version>]
                [(-c <configuration file> | --config <configuration file>)]
                [-D <system properties>...] [--git-repo-id <git repo id>]
                [--git-user-id <git user id>] [--group-id <group id>]
                [--http-user-agent <http user agent>]
                (-i <spec file> | --input-spec <spec file>)
                [--ignore-file-override <ignore file override location>]
                [--import-mappings <import mappings>...]
                [--instantiation-types <instantiation types>...]
                [--invoker-package <invoker package>]
                (-l <language> | --lang <language>)
                [--language-specific-primitives <language specific primitives>...]
                [--library <library>] [--model-name-prefix <model name prefix>]
                [--model-name-suffix <model name suffix>]
                [--model-package <model package>]
                [(-o <output directory> | --output <output directory>)]
                [--release-note <release note>] [--remove-operation-id-prefix]
                [--reserved-words-mappings <reserved word mappings>...]
                [(-s | --skip-overwrite)]
                [(-t <template directory> | --template-dir <template directory>)]
                [--type-mappings <type mappings>...] [(-v | --verbose)]
   OPTIONS
        -a <authorization>, --auth <authorization>
            adds authorization headers when fetching the swagger definitions
            remotely. Pass in a URL-encoded string of name:header with a comma
            separating multiple values
        --additional-properties <additional properties>
            sets additional properties that can be referenced by the mustache
            templates in the format of name=value,name=value. You can also have
            multiple occurrences of this option.
        --api-package <api package>
            package for generated api classes
        --artifact-id <artifact id>
            artifactId in generated pom.xml
        --artifact-version <artifact version>
            artifact version in generated pom.xml
        -c <configuration file>, --config <configuration file>
            Path to json configuration file. File content should be in a json
            format {"optionKey":"optionValue", "optionKey1":"optionValue1"...}
            Supported options can be different for each language. Run
            config-help -l {lang} command for language specific config options.
        -D <system properties>
            sets specified system properties in the format of
            name=value,name=value (or multiple options, each with name=value)
        --git-repo-id <git repo id>
            Git repo ID, e.g. swagger-codegen.
        --git-user-id <git user id>
            Git user ID, e.g. swagger-api.
        --group-id <group id>
            groupId in generated pom.xml
        --http-user-agent <http user agent>
            HTTP user agent, e.g. codegen_csharp_api_client, default to
            'Swagger-Codegen/{packageVersion}}/{language}'
        -i <spec file>, --input-spec <spec file>
            location of the swagger spec, as URL or file (required)
        --ignore-file-override <ignore file override location>
            Specifies an override location for the .swagger-codegen-ignore file.
            Most useful on initial generation.
        --import-mappings <import mappings>
            specifies mappings between a given class and the import that should
            be used for that class in the format of type=import,type=import. You
            can also have multiple occurrences of this option.
        --instantiation-types <instantiation types>
            sets instantiation type mappings in the format of
            type=instantiatedType,type=instantiatedType.For example (in Java):
            array=ArrayList,map=HashMap. In other words array types will get
            instantiated as ArrayList in generated code. You can also have
            multiple occurrences of this option.
        --invoker-package <invoker package>
            root package for generated code
        -l <language>, --lang <language>
            client language to generate (maybe class name in classpath,
            required)
        --language-specific-primitives <language specific primitives>
            specifies additional language specific primitive types in the format
            of type1,type2,type3,type3. For example:
            String,boolean,Boolean,Double. You can also have multiple
            occurrences of this option.
        --library <library>
            library template (sub-template)
        --model-name-prefix <model name prefix>
            Prefix that will be prepended to all model names. Default is the
            empty string.
        --model-name-suffix <model name suffix>
            Suffix that will be appended to all model names. Default is the
            empty string.
        --model-package <model package>
            package for generated models
        -o <output directory>, --output <output directory>
            where to write the generated files (current dir by default)
        --release-note <release note>
            Release note, default to 'Minor update'.
        --remove-operation-id-prefix
            Remove prefix of operationId, e.g. config_getId => getId
        --reserved-words-mappings <reserved word mappings>
            specifies how a reserved name should be escaped to. Otherwise, the
            default _<name> is used. For example id=identifier. You can also
            have multiple occurrences of this option.
        -s, --skip-overwrite
            specifies if the existing files should be overwritten during the
            generation.
        -t <template directory>, --template-dir <template directory>
            folder containing the template files
        --type-mappings <type mappings>
            sets mappings between swagger spec types and generated code types in
            the format of swaggerType=generatedType,swaggerType=generatedType.
            For example: array=List,map=Map,string=String. You can also have
            multiple occurrences of this option.
        -v, --verbose
            verbose mode
   ```
   
   几个主要参数：
   * -i 表示输入的文件，editor生成的设计文件路径，如：-i ~/Desktop/swagger.yaml
   * -o 代码生成目录，swagger codegen 把代码生成到什么地方，如：-o ~/Desktop
   * -l 生成代码语言，我们是生成java，如：-l java
   * -c 配置文件，配制文件路径，如：-c ~/Desktop/config.json

   最后生成代码
   
   ```
   java -jar swagger-codegen-cli.jar generate -i swagger.yaml -o client -l java -c config.json
   ```
   
   成功在 ~/Desktop 下生成了相应的 code 和 doc
   
   
   参考资料以及推荐阅读的资料  
   [http://docs.ansible.com/ansible/latest/YAMLSyntax.html](http://docs.ansible.com/ansible/latest/YAMLSyntax.html)
   [https://swagger.io/docs/specification/about/](https://swagger.io/docs/specification/about/)  
   [http://www.jianshu.com/p/c178c18aaf43](http://www.jianshu.com/p/c178c18aaf43)  
   [https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md)  
   [https://github.com/swagger-api](https://github.com/swagger-api)
   [https://www.gitbook.com/book/huangwenchao/swagger/details](https://www.gitbook.com/book/huangwenchao/swagger/details)