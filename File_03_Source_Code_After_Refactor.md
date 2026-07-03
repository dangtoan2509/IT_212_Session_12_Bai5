# Mã nguồn sau khi Refactor (Clean Code & Design Pattern)

Dưới đây là mã nguồn đã được Refactor, tách bạch Validation, chuẩn hóa Error Response và thêm Logging (SLF4J).

## 1. DTO Chuẩn hóa lỗi: ErrorResponse.java
```java
package com.abcbank.ekyc.dto;

import lombok.Builder;
import lombok.Data;

import java.time.LocalDateTime;

@Data
@Builder
public class ErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
}
```

## 2. Validator Component (Tách logic kiểm tra): AccountValidator.java
```java
package com.abcbank.ekyc.validator;

import com.abcbank.ekyc.dto.AccountRegistrationRequest;
import com.abcbank.ekyc.exception.DuplicateResourceException;
import com.abcbank.ekyc.repository.AccountRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class AccountValidator {

    private final AccountRepository accountRepository;

    public void validateRegistration(AccountRegistrationRequest request) {
        if (accountRepository.existsByCitizenId(request.getCitizenId())) {
            throw new DuplicateResourceException("Số CCCD đã tồn tại trên hệ thống!");
        }
        if (accountRepository.existsByPhone(request.getPhone())) {
            throw new DuplicateResourceException("Số điện thoại đã được đăng ký!");
        }
    }
}
```

## 3. AccountServiceImpl.java (Sau Refactor)
```java
package com.abcbank.ekyc.service.impl;

import com.abcbank.ekyc.dto.AccountRegistrationRequest;
import com.abcbank.ekyc.dto.AccountRegistrationResponse;
import com.abcbank.ekyc.entity.Account;
import com.abcbank.ekyc.entity.AccountStatus;
import com.abcbank.ekyc.repository.AccountRepository;
import com.abcbank.ekyc.service.AccountService;
import com.abcbank.ekyc.validator.AccountValidator;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.security.SecureRandom;

@Slf4j // Sử dụng SLF4J để ghi Log
@Service
@RequiredArgsConstructor
public class AccountServiceImpl implements AccountService {

    private final AccountRepository accountRepository;
    private final AccountValidator accountValidator;
    private final SecureRandom secureRandom = new SecureRandom();

    @Override
    @Transactional
    public AccountRegistrationResponse registerAccount(AccountRegistrationRequest request) {
        log.info("Bắt đầu xử lý đăng ký tài khoản cho SĐT: {}", request.getPhone());
        
        // 1. Tách logic validation
        accountValidator.validateRegistration(request);

        // 2. Build Entity
        Account account = new Account();
        account.setFullName(request.getFullName());
        account.setPhone(request.getPhone());
        account.setEmail(request.getEmail());
        account.setCitizenId(request.getCitizenId());
        account.setAccountNumber(generateAccountNumber());
        account.setStatus(AccountStatus.PENDING);

        // 3. Lưu vào cơ sở dữ liệu
        Account savedAccount = accountRepository.save(account);
        
        log.info("Tạo tài khoản thành công với ID: {}", savedAccount.getId());

        return AccountRegistrationResponse.builder()
                .accountId(savedAccount.getId())
                .accountNumber(savedAccount.getAccountNumber())
                .status(savedAccount.getStatus().name())
                .build();
    }

    private String generateAccountNumber() {
        long number = 100000000000L + (long)(secureRandom.nextDouble() * 900000000000L);
        return String.valueOf(number);
    }
}
```

## 4. AccountController.java (Sau Refactor)
```java
package com.abcbank.ekyc.controller;

import com.abcbank.ekyc.dto.AccountRegistrationRequest;
import com.abcbank.ekyc.dto.AccountRegistrationResponse;
import com.abcbank.ekyc.service.AccountService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@Slf4j
@RestController
@RequestMapping("/api/v1/accounts")
@RequiredArgsConstructor
public class AccountController {

    private final AccountService accountService;

    @PostMapping("/register")
    public ResponseEntity<AccountRegistrationResponse> register(
            @Valid @RequestBody AccountRegistrationRequest request) {
        
        log.info("Nhận request Mở tài khoản với SĐT: {}", request.getPhone());
        
        AccountRegistrationResponse response = accountService.registerAccount(request);
        
        log.info("Request xử lý thành công, trả về HTTP 201 Created");
        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }
}
```

## 5. GlobalExceptionHandler.java (Sau Refactor)
```java
package com.abcbank.ekyc.exception;

import com.abcbank.ekyc.dto.ErrorResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.context.request.WebRequest;

import java.time.LocalDateTime;
import java.util.stream.Collectors;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationExceptions(MethodArgumentNotValidException ex, WebRequest request) {
        String errors = ex.getBindingResult().getAllErrors().stream()
                .map(error -> ((FieldError) error).getField() + ": " + error.getDefaultMessage())
                .collect(Collectors.joining(", "));
        
        log.warn("Lỗi Validation request: {}", errors);

        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.BAD_REQUEST.value())
                .error(HttpStatus.BAD_REQUEST.getReasonPhrase())
                .message(errors)
                .path(request.getDescription(false).replace("uri=", ""))
                .build();

        return new ResponseEntity<>(errorResponse, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicateResource(DuplicateResourceException ex, WebRequest request) {
        log.warn("Lỗi Duplicate Resource: {}", ex.getMessage());

        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.CONFLICT.value())
                .error(HttpStatus.CONFLICT.getReasonPhrase())
                .message(ex.getMessage())
                .path(request.getDescription(false).replace("uri=", ""))
                .build();

        return new ResponseEntity<>(errorResponse, HttpStatus.CONFLICT);
    }
    
    // Gom tất cả lỗi không xác định (Internal Server Error)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(Exception ex, WebRequest request) {
        log.error("Lỗi Hệ Thống: ", ex);

        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
                .error(HttpStatus.INTERNAL_SERVER_ERROR.getReasonPhrase())
                .message("Đã xảy ra lỗi hệ thống, vui lòng thử lại sau.")
                .path(request.getDescription(false).replace("uri=", ""))
                .build();

        return new ResponseEntity<>(errorResponse, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```
