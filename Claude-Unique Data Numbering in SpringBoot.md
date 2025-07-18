# Unique Data Numbering in SpringBoot

**Created:** 7/18/2025 1:40  
**Updated:** 7/18/2025 1:43  
**Exported:** 7/18/2025 8:38  
**Link:** [https://claude.ai/chat/26a92605-2463-4351-894d-d8cfc6944ada](https://claude.ai/chat/26a92605-2463-4351-894d-d8cfc6944ada)  

## Prompt:
使用springboot+mybatisplus+xml+mysql，我需要往一个系统写入数据，name，number不能重复，同时我还要自动给一类数据指定number前缀的数据，后面依次加_100，_200，我怎么设计一张缓存表的增删改查来控制新增还有修改时的name，number不重复，以及自动根据前缀命名number



## Response:
```plaintext
Thought process: 设计数据库系统，实现唯一性验证和自动编号策略

用户想要设计一个系统，使用SpringBoot + MyBatis Plus + XML + MySQL，需要实现：
1. name和number字段不能重复
2. 自动给一类数据指定number前缀，后面依次加_100，_200等
3. 需要设计一张缓存表来控制新增和修改时的唯一性验证
4. 实现增删改查功能

我需要提供：
1. 数据库表设计
2. 实体类设计
3. Mapper接口和XML配置
4. Service层实现
5. 缓存表的设计和使用
```

我来帮你设计一个完整的解决方案，包括缓存表设计和业务逻辑实现。

**SpringBoot缓存表设计方案**

```markdown
# SpringBoot + MyBatis Plus 缓存表设计方案

## 1. 数据库表设计

### 1.1 业务表 (business_data)
```sql
CREATE TABLE `business_data` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(100) NOT NULL COMMENT '名称',
  `number` varchar(100) NOT NULL COMMENT '编号',
  `data_type` varchar(50) DEFAULT NULL COMMENT '数据类型',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_name` (`name`),
  UNIQUE KEY `uk_number` (`number`),
  KEY `idx_data_type` (`data_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='业务数据表';
```

### 1.2 缓存表 (number_sequence_cache)
```sql
CREATE TABLE `number_sequence_cache` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `prefix` varchar(50) NOT NULL COMMENT '编号前缀',
  `current_sequence` int(11) NOT NULL DEFAULT 0 COMMENT '当前序列号',
  `step` int(11) NOT NULL DEFAULT 100 COMMENT '步长',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_prefix` (`prefix`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='编号序列缓存表';
```

## 2. 实体类设计

### 2.1 BusinessData实体类
```java
import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.time.LocalDateTime;

@Data
@TableName("business_data")
public class BusinessData {
    @TableId(type = IdType.AUTO)
    private Long id;
    
    private String name;
    
    private String number;
    
    private String dataType;
    
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
    
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
}
```

### 2.2 NumberSequenceCache实体类
```java
import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.time.LocalDateTime;

@Data
@TableName("number_sequence_cache")
public class NumberSequenceCache {
    @TableId(type = IdType.AUTO)
    private Long id;
    
    private String prefix;
    
    private Integer currentSequence;
    
    private Integer step;
    
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
    
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
}
```

## 3. Mapper接口和XML

### 3.1 BusinessDataMapper
```java
import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

@Mapper
public interface BusinessDataMapper extends BaseMapper<BusinessData> {
    
    /**
     * 检查名称是否存在（排除指定ID）
     */
    Integer checkNameExists(@Param("name") String name, @Param("excludeId") Long excludeId);
    
    /**
     * 检查编号是否存在（排除指定ID）
     */
    Integer checkNumberExists(@Param("number") String number, @Param("excludeId") Long excludeId);
}
```

### 3.2 BusinessDataMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.BusinessDataMapper">
    
    <select id="checkNameExists" resultType="java.lang.Integer">
        SELECT COUNT(1) FROM business_data 
        WHERE name = #{name}
        <if test="excludeId != null">
            AND id != #{excludeId}
        </if>
    </select>
    
    <select id="checkNumberExists" resultType="java.lang.Integer">
        SELECT COUNT(1) FROM business_data 
        WHERE number = #{number}
        <if test="excludeId != null">
            AND id != #{excludeId}
        </if>
    </select>
    
</mapper>
```

### 3.3 NumberSequenceCacheMapper
```java
import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

@Mapper
public interface NumberSequenceCacheMapper extends BaseMapper<NumberSequenceCache> {
    
    /**
     * 获取并更新序列号（使用悲观锁）
     */
    NumberSequenceCache getAndUpdateSequence(@Param("prefix") String prefix);
    
    /**
     * 更新序列号
     */
    int updateSequence(@Param("prefix") String prefix, @Param("newSequence") Integer newSequence);
}
```

### 3.4 NumberSequenceCacheMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.NumberSequenceCacheMapper">
    
    <select id="getAndUpdateSequence" resultType="com.example.entity.NumberSequenceCache">
        SELECT * FROM number_sequence_cache 
        WHERE prefix = #{prefix}
        FOR UPDATE
    </select>
    
    <update id="updateSequence">
        UPDATE number_sequence_cache 
        SET current_sequence = #{newSequence},
            update_time = NOW()
        WHERE prefix = #{prefix}
    </update>
    
</mapper>
```

## 4. Service层实现

### 4.1 BusinessDataService接口
```java
import com.baomidou.mybatisplus.extension.service.IService;

public interface BusinessDataService extends IService<BusinessData> {
    
    /**
     * 新增业务数据
     */
    boolean saveBusinessData(BusinessData businessData);
    
    /**
     * 更新业务数据
     */
    boolean updateBusinessData(BusinessData businessData);
    
    /**
     * 根据前缀生成编号
     */
    String generateNumber(String prefix);
}
```

### 4.2 BusinessDataServiceImpl实现类
```java
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.StringUtils;

@Service
public class BusinessDataServiceImpl extends ServiceImpl<BusinessDataMapper, BusinessData> 
        implements BusinessDataService {
    
    @Autowired
    private NumberSequenceCacheMapper sequenceCacheMapper;
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean saveBusinessData(BusinessData businessData) {
        // 验证name唯一性
        if (checkNameExists(businessData.getName(), null)) {
            throw new RuntimeException("名称已存在");
        }
        
        // 如果没有指定number，根据dataType自动生成
        if (!StringUtils.hasText(businessData.getNumber())) {
            String prefix = businessData.getDataType();
            if (StringUtils.hasText(prefix)) {
                businessData.setNumber(generateNumber(prefix));
            }
        } else {
            // 验证number唯一性
            if (checkNumberExists(businessData.getNumber(), null)) {
                throw new RuntimeException("编号已存在");
            }
        }
        
        return this.save(businessData);
    }
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean updateBusinessData(BusinessData businessData) {
        if (businessData.getId() == null) {
            throw new RuntimeException("ID不能为空");
        }
        
        // 验证name唯一性
        if (StringUtils.hasText(businessData.getName()) && 
            checkNameExists(businessData.getName(), businessData.getId())) {
            throw new RuntimeException("名称已存在");
        }
        
        // 验证number唯一性
        if (StringUtils.hasText(businessData.getNumber()) && 
            checkNumberExists(businessData.getNumber(), businessData.getId())) {
            throw new RuntimeException("编号已存在");
        }
        
        return this.updateById(businessData);
    }
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public String generateNumber(String prefix) {
        // 获取或创建序列缓存记录
        NumberSequenceCache cache = sequenceCacheMapper.getAndUpdateSequence(prefix);
        
        if (cache == null) {
            // 首次使用该前缀，创建新记录
            cache = new NumberSequenceCache();
            cache.setPrefix(prefix);
            cache.setCurrentSequence(100);
            cache.setStep(100);
            sequenceCacheMapper.insert(cache);
            return prefix + "_100";
        }
        
        // 计算新的序列号
        Integer newSequence = cache.getCurrentSequence() + cache.getStep();
        
        // 更新序列号
        sequenceCacheMapper.updateSequence(prefix, newSequence);
        
        return prefix + "_" + newSequence;
    }
    
    private boolean checkNameExists(String name, Long excludeId) {
        return baseMapper.checkNameExists(name, excludeId) > 0;
    }
    
    private boolean checkNumberExists(String number, Long excludeId) {
        return baseMapper.checkNumberExists(number, excludeId) > 0;
    }
}
```

## 5. Controller层示例

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/business-data")
public class BusinessDataController {
    
    @Autowired
    private BusinessDataService businessDataService;
    
    @PostMapping("/save")
    public Result<Boolean> save(@RequestBody BusinessData businessData) {
        try {
            boolean result = businessDataService.saveBusinessData(businessData);
            return Result.success(result);
        } catch (Exception e) {
            return Result.error(e.getMessage());
        }
    }
    
    @PutMapping("/update")
    public Result<Boolean> update(@RequestBody BusinessData businessData) {
        try {
            boolean result = businessDataService.updateBusinessData(businessData);
            return Result.success(result);
        } catch (Exception e) {
            return Result.error(e.getMessage());
        }
    }
    
    @GetMapping("/generateNumber")
    public Result<String> generateNumber(@RequestParam String prefix) {
        try {
            String number = businessDataService.generateNumber(prefix);
            return Result.success(number);
        } catch (Exception e) {
            return Result.error(e.getMessage());
        }
    }
}
```

## 6. 配置文件示例

### application.yml
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/your_database?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8
    username: root
    password: your_password
    driver-class-name: com.mysql.cj.jdbc.Driver

mybatis-plus:
  mapper-locations: classpath:mapper/*.xml
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

## 7. 使用说明

1. **自动编号生成**：当创建数据时，如果没有指定number，系统会根据dataType字段作为前缀自动生成编号。

2. **手动指定编号**：也可以手动指定number，系统会验证其唯一性。

3. **并发控制**：使用数据库的悲观锁（FOR UPDATE）来保证并发情况下编号生成的唯一性。

4. **缓存表维护**：缓存表会自动记录每个前缀的当前序列号，无需手动维护。

5. **扩展性**：可以根据需要调整步长（step）或添加其他规则。

## 8. 优化建议

1. **Redis缓存**：对于高并发场景，可以使用Redis来管理序列号，减少数据库压力。

2. **批量生成**：可以一次性生成多个编号，减少数据库交互。

3. **异步处理**：对于非关键业务，可以使用消息队列异步生成编号。

4. **定期清理**：可以添加定时任务清理长期未使用的缓存记录。
```

```plaintext
Thought process: 设计全面的数据库方案，确保唯一性和自动编号

用户需要一个完整的方案来处理name和number的唯一性验证，以及自动生成带前缀的编号。我已经提供了一个完整的解决方案，包括：

1. 数据库表设计（业务表和缓存表）
2. 实体类
3. Mapper接口和XML
4. Service层实现（包含完整的业务逻辑）
5. Controller示例
6. 配置文件
7. 使用说明和优化建议

这个方案的核心特点：
- 使用数据库唯一索引保证name和number的唯一性
- 使用缓存表来管理编号序列
- 使用悲观锁保证并发安全
- 支持自动生成和手动指定两种方式
- 提供了完整的增删改查功能

我觉得这个方案应该能满足用户的需求。
```

我为你设计了一个完整的缓存表方案来控制数据的唯一性和自动编号生成。这个方案包括：

## 核心设计思路：

1. **业务表（business_data）**：存储实际业务数据，name和number字段设置唯一索引
2. **缓存表（number_sequence_cache）**：管理各个前缀的序列号，记录当前序列值

## 主要功能特点：

### 1. 唯一性控制
- 数据库层面：通过唯一索引保证name和number不重复
- 应用层面：在新增和修改前进行唯一性校验

### 2. 自动编号生成
- 根据数据类型（dataType）作为前缀
- 自动追加序列号（_100、_200、_300...）
- 使用悲观锁保证并发安全

### 3. 灵活性
- 支持自动生成编号
- 也支持手动指定编号
- 可配置步长（默认100）

## 使用示例：

```java
// 1. 自动生成编号
BusinessData data = new BusinessData();
data.setName("测试数据1");
data.setDataType("ORDER"); // 将自动生成 ORDER_100, ORDER_200...
businessDataService.saveBusinessData(data);

// 2. 手动指定编号
BusinessData data2 = new BusinessData();
data2.setName("测试数据2");
data2.setNumber("CUSTOM_001");
businessDataService.saveBusinessData(data2);
```

这个方案的优势：
- 事务保证数据一致性
- 悲观锁防止并发冲突
- 缓存表独立管理序列号，易于维护
- 支持多种前缀并行使用

你还可以根据实际需求进行扩展，比如添加Redis缓存来提升性能，或者实现批量编号预生成等功能。





---
Powered by [Claude Exporter](https://www.claudexporter.com)