# Mã nguồn trước khi Refactor (Bài 3)

Dưới đây là các class chính từ Bài 3 cần được Refactor:

## AccountServiceImpl.java
```java
package com.abcbank.ekyc.service.impl;

import com.abcbank.ekyc.dto.AccountRegistrationRequest;
import com.abcbank.ekyc.dto.AccountRegistrationResponse;
import com.abcbank.ekyc.entity.Account;
import com.abcbank.ekyc.entity.AccountStatus;
import com.abcbank.ekyc.exception.DuplicateResourceException;
import com.abcbank.ekyc.repository.AccountRepository;
import com.abcbank.ekyc.service.AccountService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.security.SecureRandom;

@Service
@RequiredArgsConstructor
public class AccountServiceImpl implements AccountService {

    private final AccountRepository accountRepository;
    private final SecureRandom secureRandom = new SecureRandom();

    @Override
    @Transactional
    public AccountRegistrationResponse registerAccount(AccountRegistrationRequest request) {
        
        // CODE SMELL: Logic validation bị nhồi nhét vào Service chính (Vi phạm Single Responsibility)
        if (accountRepository.existsByCitizenId(request.getCitizenId())) {
            throw new DuplicateResourceException("Số CCCD đã tồn tại trên hệ thống!");
        }
        if (accountRepository.existsByPhone(request.getPhone())) {
            throw new DuplicateResourceException("Số điện thoại đã được đăng ký!");
        }

        // TẠO ENTITY
        Account account = new Account();
        account.setFullName(request.getFullName());
        account.setPhone(request.getPhone());
        account.setEmail(request.getEmail());
        account.setCitizenId(request.getCitizenId());
        account.setAccountNumber(generateAccountNumber());
        account.setStatus(AccountStatus.PENDING);

        Account savedAccount = accountRepository.save(account);

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

## GlobalExceptionHandler.java
```java
package com.abcbank.ekyc.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

import java.util.HashMap;
import java.util.Map;

@ControllerAdvice
public class GlobalExceptionHandler {

    // CODE SMELL: Trả về Map<String, String> thiếu chuẩn hóa. Không có thông tin timestamp, path, hay status code rõ ràng.
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationExceptions(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach((error) -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<Map<String, String>> handleDuplicateResource(DuplicateResourceException ex) {
        Map<String, String> error = new HashMap<>();
        error.put("error", ex.getMessage());
        return new ResponseEntity<>(error, HttpStatus.CONFLICT); 
    }
}
```

## AccountController.java
```java
package com.abcbank.ekyc.controller;

import com.abcbank.ekyc.dto.AccountRegistrationRequest;
import com.abcbank.ekyc.dto.AccountRegistrationResponse;
import com.abcbank.ekyc.service.AccountService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/accounts")
@RequiredArgsConstructor
public class AccountController {

    private final AccountService accountService;

    // CODE SMELL: Thiếu Logging (Không biết request bắt đầu và kết thúc như thế nào trên môi trường thật)
    @PostMapping("/register")
    public ResponseEntity<AccountRegistrationResponse> register(
            @Valid @RequestBody AccountRegistrationRequest request) {
        
        AccountRegistrationResponse response = accountService.registerAccount(request);
        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }
}
```
