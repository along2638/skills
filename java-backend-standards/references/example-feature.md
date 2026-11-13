# 完整示例：订单功能垂直切片

本示例覆盖一张表从 DDL 到 Controller 的完整实现，可直接作为新增业务域的模板。基线：Spring Boot 2.7.18、JDK 8、MyBatis-Plus 3.5.7、MySQL 8.0。

示例为聚焦业务逻辑，省略了部分 Javadoc（类注释、`@author`/`@date`、接口方法说明）。落地时按 `java-coding.md` §3 补齐：类注释写职责，Service 接口方法与 Controller 方法写清用途、参数、异常。

示例中 `OrderEntity` 对应 PO，`OrderCreateReq` / `OrderQueryReq` 对应 DTO，`OrderResp` 对应 VO。

```text
src/main/java/com/acme/mall
├── common/web/UserContext.java
├── controller/OrderController.java
├── service/OrderService.java
├── service/impl/OrderServiceImpl.java
├── mapper/OrderMapper.java
├── entity/OrderEntity.java
├── enums/OrderStatus.java
├── enums/OrderErrorCode.java
├── dto/OrderCreateReq.java
├── dto/OrderQueryReq.java
├── vo/OrderResp.java
└── convert/OrderConvert.java

src/main/resources
├── mapper/OrderMapper.xml
└── db/migration/V1__init_trade_order.sql
```

这是单业务域的扁平分包；出现第二个业务域时，把上述业务包整体迁入 `modules/{domain}`。

## 1. 表结构

```sql
CREATE TABLE `trade_order` (
  `id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键',
  `order_no` VARCHAR(32) NOT NULL COMMENT '订单号，全局唯一且不复用',
  `user_id` BIGINT NOT NULL COMMENT '下单用户ID',
  `amount` DECIMAL(18,2) NOT NULL DEFAULT 0.00 COMMENT '订单金额，单位元',
  `status` VARCHAR(32) NOT NULL DEFAULT 'WAIT_PAY'
      COMMENT '状态：WAIT_PAY待支付 PAID已支付 CANCELED已取消 COMPLETED已完成',
  `remark` VARCHAR(500) NOT NULL DEFAULT '' COMMENT '备注',
  `create_time` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
  `update_time` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)
      ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
  `create_by` BIGINT NOT NULL DEFAULT 0 COMMENT '创建人ID，0为系统',
  `update_by` BIGINT NOT NULL DEFAULT 0 COMMENT '更新人ID，0为系统',
  `deleted` TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除 1已删除',
  `version` INT NOT NULL DEFAULT 0 COMMENT '乐观锁版本',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_trade_order_order_no` (`order_no`),
  KEY `idx_trade_order_user_status_time` (`user_id`, `status`, `create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='交易订单';
```

## 2. 枚举

```java
package com.acme.mall.enums;

import com.baomidou.mybatisplus.annotation.IEnum;
import com.fasterxml.jackson.annotation.JsonValue;

public enum OrderStatus implements IEnum<String> {

    WAIT_PAY("WAIT_PAY", "待支付"),
    PAID("PAID", "已支付"),
    CANCELED("CANCELED", "已取消"),
    COMPLETED("COMPLETED", "已完成");

    private final String code;
    private final String desc;

    OrderStatus(String code, String desc) {
        this.code = code;
        this.desc = desc;
    }

    @Override
    @JsonValue
    public String getValue() {
        return code;
    }

    public String getDesc() {
        return desc;
    }
}
```

```java
package com.acme.mall.enums;

import com.acme.mall.common.api.ErrorCode;
import org.springframework.http.HttpStatus;

public enum OrderErrorCode implements ErrorCode {

    ORDER_NOT_FOUND(30101, "订单不存在", HttpStatus.NOT_FOUND),
    ORDER_STATUS_NOT_ALLOWED(30102, "当前订单状态不允许该操作", HttpStatus.CONFLICT),
    ORDER_NO_DUPLICATED(30103, "订单号已存在", HttpStatus.CONFLICT),
    ORDER_CONCURRENT_MODIFIED(30104, "订单已被修改，请刷新后重试", HttpStatus.CONFLICT);

    private final int code;
    private final String message;
    private final HttpStatus httpStatus;

    OrderErrorCode(int code, String message, HttpStatus httpStatus) {
        this.code = code;
        this.message = message;
        this.httpStatus = httpStatus;
    }

    @Override
    public int getCode() {
        return code;
    }

    @Override
    public String getMessage() {
        return message;
    }

    @Override
    public HttpStatus getHttpStatus() {
        return httpStatus;
    }
}
```

## 3. Entity

```java
package com.acme.mall.entity;

import com.acme.mall.enums.OrderStatus;
import com.baomidou.mybatisplus.annotation.FieldFill;
import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableLogic;
import com.baomidou.mybatisplus.annotation.TableName;
import com.baomidou.mybatisplus.annotation.Version;
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Getter
@Setter
@ToString
@TableName("trade_order")
public class OrderEntity {

    @TableId(type = IdType.AUTO)
    private Long id;

    private String orderNo;

    private Long userId;

    private BigDecimal amount;

    private OrderStatus status;

    private String remark;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableField(fill = FieldFill.INSERT)
    private Long createBy;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private Long updateBy;

    @TableLogic
    private Integer deleted;

    @Version
    private Integer version;
}
```

## 4. Mapper

```java
package com.acme.mall.mapper;

import com.acme.mall.entity.OrderEntity;
import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

@Mapper
public interface OrderMapper extends BaseMapper<OrderEntity> {

    OrderEntity selectByOrderNo(@Param("orderNo") String orderNo);
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.acme.mall.mapper.OrderMapper">

    <sql id="Base_Column_List">
        id, order_no, user_id, amount, status, remark,
        create_time, update_time, create_by, update_by, deleted, version
    </sql>

    <select id="selectByOrderNo" resultType="com.acme.mall.entity.OrderEntity">
        SELECT <include refid="Base_Column_List"/>
        FROM trade_order
        WHERE order_no = #{orderNo}
          AND deleted = 0
        LIMIT 1
    </select>
</mapper>
```

XML 手写 SQL 不会自动追加逻辑删除条件，必须显式写 `deleted = 0`。枚举字段依赖 `default-enum-type-handler`，配置见 `tech-stack.md`。

## 5. 请求与响应对象

```java
package com.acme.mall.dto;

import lombok.Data;

import javax.validation.constraints.DecimalMin;
import javax.validation.constraints.Digits;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Size;
import java.math.BigDecimal;

@Data
public class OrderCreateReq {

    @NotNull(message = "金额不能为空")
    @DecimalMin(value = "0.01", message = "金额必须大于0")
    @Digits(integer = 16, fraction = 2, message = "金额最多两位小数")
    private BigDecimal amount;

    @Size(max = 500, message = "备注最多500字")
    private String remark;
}
```

`userId` 不放在请求体，由登录上下文获取，防止用户伪造他人身份下单。

```java
package com.acme.mall.dto;

import com.acme.mall.common.api.PageReq;
import com.acme.mall.enums.OrderStatus;
import lombok.Data;
import lombok.EqualsAndHashCode;
import org.springframework.format.annotation.DateTimeFormat;

import java.time.LocalDateTime;

@Data
@EqualsAndHashCode(callSuper = true)
public class OrderQueryReq extends PageReq {

    private OrderStatus status;

    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime startTime;

    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime endTime;
}
```

```java
package com.acme.mall.vo;

import com.acme.mall.enums.OrderStatus;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import com.fasterxml.jackson.databind.ser.std.ToStringSerializer;
import lombok.Data;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
public class OrderResp {

    @JsonSerialize(using = ToStringSerializer.class)
    private Long id;

    private String orderNo;

    @JsonSerialize(using = ToStringSerializer.class)
    private Long userId;

    private BigDecimal amount;

    private OrderStatus status;

    private String remark;

    private LocalDateTime createTime;

    private LocalDateTime updateTime;
}
```

## 6. Hutool 转换

```java
package com.acme.mall.convert;

import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.bean.copier.CopyOptions;
import com.acme.mall.dto.OrderCreateReq;
import com.acme.mall.entity.OrderEntity;
import com.acme.mall.vo.OrderResp;
import org.springframework.stereotype.Component;

@Component
public class OrderConvert {

    private static final CopyOptions CREATE_REQ_TO_ENTITY = CopyOptions.create()
            .setIgnoreProperties("id", "orderNo", "userId", "status",
                    "createTime", "updateTime", "createBy", "updateBy", "deleted", "version");

    public OrderEntity toEntity(OrderCreateReq req) {
        return BeanUtil.toBean(req, OrderEntity.class, CREATE_REQ_TO_ENTITY);
    }

    public OrderResp toResp(OrderEntity entity) {
        return entity == null ? null : BeanUtil.toBean(entity, OrderResp.class);
    }
}
```

入参转 Entity 通过 `CopyOptions` 忽略主键、归属用户、状态与审计字段；`userId`、`status` 等由 Service 根据登录上下文和业务规则赋值，不能被请求体覆盖。

## 7. 登录上下文

```java
package com.acme.mall.common.web;

import com.acme.mall.common.api.CommonErrorCode;
import com.acme.mall.common.exception.BizException;
import org.springframework.security.authentication.AnonymousAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;

public final class UserContext {

    private UserContext() {
    }

    public static Long requireUserId() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication == null
                || !authentication.isAuthenticated()
                || authentication instanceof AnonymousAuthenticationToken) {
            throw new BizException(CommonErrorCode.UNAUTHORIZED);
        }
        try {
            return Long.valueOf(authentication.getName());
        } catch (NumberFormatException e) {
            throw new BizException(CommonErrorCode.UNAUTHORIZED);
        }
    }
}
```

JWT 过滤器必须把 `subject`（用户 ID）设置为 `Authentication.getName()`。未使用 Spring Security 的项目，替换为自身登录上下文，但**不要信任请求体里的 userId**。

## 8. Service

业务 Service 强制接口 + 实现：`OrderService` 放 `service`，`OrderServiceImpl` 放 `service/impl`；Controller 只注入接口。下面的 `OrderServiceImpl` 是业务实现类，不继承 MyBatis-Plus 的 `ServiceImpl`。

```java
package com.acme.mall.service;

import com.acme.mall.common.api.PageResp;
import com.acme.mall.dto.OrderCreateReq;
import com.acme.mall.dto.OrderQueryReq;
import com.acme.mall.vo.OrderResp;

public interface OrderService {

    Long create(OrderCreateReq req);

    OrderResp get(Long id);

    PageResp<OrderResp> page(OrderQueryReq req);

    void cancel(Long id);
}
```

```java
package com.acme.mall.service.impl;

import com.acme.mall.common.api.PageResp;
import com.acme.mall.common.exception.BizException;
import com.acme.mall.common.web.UserContext;
import com.acme.mall.convert.OrderConvert;
import com.acme.mall.dto.OrderCreateReq;
import com.acme.mall.dto.OrderQueryReq;
import com.acme.mall.entity.OrderEntity;
import com.acme.mall.enums.OrderErrorCode;
import com.acme.mall.enums.OrderStatus;
import com.acme.mall.mapper.OrderMapper;
import com.acme.mall.service.OrderService;
import com.acme.mall.vo.OrderResp;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.toolkit.IdWorker;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderServiceImpl implements OrderService {

    private final OrderMapper orderMapper;
    private final OrderConvert orderConvert;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long create(OrderCreateReq req) {
        Long userId = UserContext.requireUserId();
        OrderEntity entity = orderConvert.toEntity(req);
        entity.setUserId(userId);
        entity.setOrderNo("O" + IdWorker.getIdStr());
        entity.setStatus(OrderStatus.WAIT_PAY);

        try {
            orderMapper.insert(entity);
        } catch (DuplicateKeyException e) {
            throw new BizException(OrderErrorCode.ORDER_NO_DUPLICATED);
        }

        log.info("order created, orderId={}, orderNo={}, userId={}",
                entity.getId(), entity.getOrderNo(), userId);
        return entity.getId();
    }

    @Override
    public OrderResp get(Long id) {
        Long userId = UserContext.requireUserId();
        OrderEntity entity = orderMapper.selectById(id);
        if (entity == null || !userId.equals(entity.getUserId())) {
            throw new BizException(OrderErrorCode.ORDER_NOT_FOUND);
        }
        return orderConvert.toResp(entity);
    }

    @Override
    public PageResp<OrderResp> page(OrderQueryReq req) {
        Long userId = UserContext.requireUserId();
        LambdaQueryWrapper<OrderEntity> wrapper = Wrappers.<OrderEntity>lambdaQuery()
                .eq(OrderEntity::getUserId, userId)
                .eq(req.getStatus() != null, OrderEntity::getStatus, req.getStatus())
                .ge(req.getStartTime() != null, OrderEntity::getCreateTime, req.getStartTime())
                .lt(req.getEndTime() != null, OrderEntity::getCreateTime, req.getEndTime())
                .orderByDesc(OrderEntity::getCreateTime)
                .orderByDesc(OrderEntity::getId);

        Page<OrderEntity> page = orderMapper.selectPage(
                new Page<>(req.getPageNum(), req.getPageSize()), wrapper);

        List<OrderResp> records = page.getRecords().stream()
                .map(orderConvert::toResp)
                .collect(Collectors.toList());
        return PageResp.of(records, page.getTotal(), page.getCurrent(), page.getSize());
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void cancel(Long id) {
        Long userId = UserContext.requireUserId();
        OrderEntity entity = orderMapper.selectById(id);
        if (entity == null || !userId.equals(entity.getUserId())) {
            throw new BizException(OrderErrorCode.ORDER_NOT_FOUND);
        }
        if (entity.getStatus() != OrderStatus.WAIT_PAY) {
            throw new BizException(OrderErrorCode.ORDER_STATUS_NOT_ALLOWED);
        }

        OrderEntity update = new OrderEntity();
        update.setId(id);
        update.setStatus(OrderStatus.CANCELED);
        update.setVersion(entity.getVersion());
        int rows = orderMapper.updateById(update);
        if (rows == 0) {
            throw new BizException(OrderErrorCode.ORDER_CONCURRENT_MODIFIED);
        }

        log.info("order canceled, orderId={}, userId={}", id, userId);
    }
}
```

## 9. Controller

```java
package com.acme.mall.controller;

import com.acme.mall.common.api.PageResp;
import com.acme.mall.common.api.Result;
import com.acme.mall.dto.OrderCreateReq;
import com.acme.mall.dto.OrderQueryReq;
import com.acme.mall.service.OrderService;
import com.acme.mall.vo.OrderResp;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.validation.Valid;
import javax.validation.constraints.Positive;

@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Validated
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    @PreAuthorize("hasAuthority('order:create')")
    public Result<String> create(@Valid @RequestBody OrderCreateReq req) {
        return Result.success(String.valueOf(orderService.create(req)));
    }

    @GetMapping("/{id}")
    @PreAuthorize("hasAuthority('order:view')")
    public Result<OrderResp> get(@PathVariable @Positive Long id) {
        return Result.success(orderService.get(id));
    }

    @GetMapping
    @PreAuthorize("hasAuthority('order:view')")
    public Result<PageResp<OrderResp>> page(@Valid OrderQueryReq req) {
        return Result.success(orderService.page(req));
    }

    @PostMapping("/{id}/cancel")
    @PreAuthorize("hasAuthority('order:cancel')")
    public Result<Void> cancel(@PathVariable @Positive Long id) {
        orderService.cancel(id);
        return Result.success();
    }
}
```

## 10. 调用链与约束

```text
POST /api/v1/orders
  -> Spring Security 校验 JWT 与 order:create 权限
  -> @Valid 校验 amount / remark
  -> UserContext 从 Authentication 取 userId
  -> Service 生成 orderNo、设置 WAIT_PAY
  -> INSERT（唯一索引兜底订单号）
  -> Result<String> 返回字符串 ID
```

- 事务边界在 Service 写方法；Controller 不开启事务。
- `createBy/updateBy` 需在 `MetaObjectHandler` 中从登录上下文填充；示例只演示时间字段填充，未启用时由数据库默认值 0 兜底。
- 查询/取消同时校验 `id + userId`，他人订单一律返回"订单不存在"，避免枚举。
- 取消使用 `@Version` 乐观锁，影响行数为 0 时返回 409。
- 唯一索引冲突翻译为 `30103`，不向前端暴露数据库错误。
- XML 手写查询显式过滤 `deleted = 0`。
- 金额 `BigDecimal` 出参为字符串，ID 出参为字符串。
- 时间格式、时区、枚举 code 由统一配置保证。

## 11. 上线前必须补齐

1. **幂等**：创建订单按 `Idempotency-Key` 做 Redis 幂等，不能只靠唯一订单号。
2. **权限数据**：为用户授予 `order:create/view/cancel` 权限码。
3. **测试**：按 `testing.md` 增加 Service 单元测试、Controller 切片测试、Mapper/MySQL 集成测试。
4. **审计**：订单创建、取消写审计日志。
5. **后台接口**：管理端查询另建 `/api/v1/admin/orders`，单独鉴权，不复用用户端接口。
