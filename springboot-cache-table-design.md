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