    # IT212_SS5_HW03 - Đọc hiểu & Dò lỗi qua Prompt

// Phân tích lỗ hổng của mã nguồn ban đầu

Đoạn code ban đầu có thể “chạy được” trong trường hợp dữ liệu đẹp, nhưng rất nguy hiểm nếu đưa lên môi trường Production.

Thứ nhất, code không kiểm tra dữ liệu đầu vào. Nếu `fromAccountId`, `toAccountId` bị null, hoặc `amount <= 0`, chương trình vẫn tiếp tục xử lý và có thể gây lỗi hoặc tạo giao dịch sai nghiệp vụ.

Thứ hai, code dùng `orElse(null)` khi tìm tài khoản. Nếu không tìm thấy tài khoản gửi hoặc tài khoản nhận, biến `from` hoặc `to` sẽ là null. Khi gọi `from.setBalance()` hoặc `to.setBalance()`, chương trình sẽ bị `NullPointerException`.

Thứ ba, code không kiểm tra số dư tài khoản nguồn. Nếu số dư không đủ, hệ thống vẫn trừ tiền, dẫn đến tài khoản bị âm.

Thứ tư, code không có `@Transactional`. Nếu lưu tài khoản gửi thành công nhưng lưu tài khoản nhận thất bại, dữ liệu sẽ bị mất tính nhất quán. Trong nghiệp vụ ngân hàng, giao dịch chuyển tiền phải đảm bảo tính nguyên tử: hoặc cả hai thao tác cùng thành công, hoặc tất cả phải rollback.

Thứ năm, code không có logging. Khi giao dịch thất bại hoặc có tranh chấp, hệ thống khó truy vết được tài khoản nào chuyển tiền, số tiền bao nhiêu, lỗi xảy ra ở đâu.

Cuối cùng, code dùng `double` cho tiền tệ. Trong thực tế, tiền nên dùng `BigDecimal` để tránh sai số làm tròn. Tuy nhiên trong phạm vi bài này, trọng tâm cải tiến là transaction, validation, exception và logging.

// Prompt vòng 1 - Robustness

```text
Bạn là một Java Backend Developer chuyên xử lý nghiệp vụ tài chính.

Hãy cải tiến đoạn code chuyển tiền dưới đây theo hướng Robustness.

Yêu cầu:
1. Kiểm tra fromAccountId và toAccountId không được null.
2. Kiểm tra amount phải lớn hơn 0.
3. Nếu không tìm thấy tài khoản gửi hoặc tài khoản nhận, ném custom exception AccountNotFoundException.
4. Nếu số dư tài khoản gửi không đủ, ném custom exception InsufficientBalanceException.
5. Không để xảy ra NullPointerException.
6. Trả về mã nguồn Java đã sửa gồm AccountService và các Custom Exception cần thiết.

Code ban đầu:

public class AccountService {

    private AccountRepository accountRepository;

    public void transfer(Long fromAccountId, Long toAccountId, double amount) {

        Account from = accountRepository.findById(fromAccountId).orElse(null);

        Account to = accountRepository.findById(toAccountId).orElse(null);

        from.setBalance(from.getBalance() - amount);

        to.setBalance(to.getBalance() + amount);

        accountRepository.save(from);

        accountRepository.save(to);
    }
}
```

// Prompt vòng 2 - Maintainability / Clean Code

```text
Dựa trên phiên bản code đã cải tiến ở vòng 1, hãy tiếp tục nâng cấp theo hướng Maintainability và Clean Code trong Spring Boot.

Yêu cầu:
1. Thêm @Service cho AccountService.
2. Thêm @Transactional cho phương thức transfer để đảm bảo tính ACID, đặc biệt là Atomicity.
3. Sử dụng Lombok với @RequiredArgsConstructor để inject AccountRepository qua constructor.
4. Sử dụng @Slf4j để log thông tin giao dịch.
5. Log khi bắt đầu chuyển tiền, khi chuyển tiền thành công và khi có lỗi nghiệp vụ.
6. Tách các bước kiểm tra thành method riêng để code dễ đọc và dễ bảo trì.
7. Không nuốt exception. Nếu lỗi xảy ra, phải throw lại để transaction rollback.
```

// Prompt vòng 3 - Context-specific Tuning

```text
Tiếp tục cải tiến phiên bản code ở vòng 2 cho phù hợp hơn với dự án thực tế.

Yêu cầu:
1. Phương thức transfer không trả về void nữa mà trả về TransactionResult.
2. TransactionResult gồm các thông tin:
   - success
   - message
   - fromAccountId
   - toAccountId
   - amount
3. Nếu chuyển tiền thành công, trả về TransactionResult success = true.
4. Nếu thất bại do lỗi nghiệp vụ thì exception vẫn được ném ra để rollback transaction.
5. Viết thêm JUnit 5 test case cho trường hợp chuyển tiền thất bại do không đủ số dư.
6. Test cần dùng Mockito để mock AccountRepository.
7. Trả về code Java hoàn chỉnh gồm:
   - AccountService
   - TransactionResult
   - Custom Exceptions
   - JUnit test case
```

// Mã nguồn Java đã tối ưu do AI sinh ra

```java
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class AccountService {

    private final AccountRepository accountRepository;

    @Transactional
    public TransactionResult transfer(
            Long fromAccountId,
            Long toAccountId,
            double amount
    ) {

        log.info(
                "Start transferring money from account {} to account {}, amount: {}",
                fromAccountId,
                toAccountId,
                amount
        );

        try {
            validateInput(fromAccountId, toAccountId, amount);

            Account fromAccount = findAccountOrThrow(fromAccountId);
            Account toAccount = findAccountOrThrow(toAccountId);

            validateBalance(fromAccount, amount);

            fromAccount.setBalance(fromAccount.getBalance() - amount);
            toAccount.setBalance(toAccount.getBalance() + amount);

            accountRepository.save(fromAccount);
            accountRepository.save(toAccount);

            log.info(
                    "Transfer completed successfully from account {} to account {}, amount: {}",
                    fromAccountId,
                    toAccountId,
                    amount
            );

            return new TransactionResult(
                    true,
                    "Transfer completed successfully",
                    fromAccountId,
                    toAccountId,
                    amount
            );

        } catch (RuntimeException exception) {

            log.error(
                    "Transfer failed from account {} to account {}, amount: {}, reason: {}",
                    fromAccountId,
                    toAccountId,
                    amount,
                    exception.getMessage()
            );

            throw exception;
        }
    }

    private void validateInput(
            Long fromAccountId,
            Long toAccountId,
            double amount
    ) {

        if (fromAccountId == null) {
            throw new InvalidTransferException(
                    "From account id must not be null"
            );
        }

        if (toAccountId == null) {
            throw new InvalidTransferException(
                    "To account id must not be null"
            );
        }

        if (fromAccountId.equals(toAccountId)) {
            throw new InvalidTransferException(
                    "From account and to account must be different"
            );
        }

        if (amount <= 0) {
            throw new InvalidTransferException(
                    "Transfer amount must be greater than 0"
            );
        }
    }

    private Account findAccountOrThrow(Long accountId) {

        return accountRepository.findById(accountId)
                .orElseThrow(
                        () -> new AccountNotFoundException(
                                "Account not found with id: " + accountId
                        )
                );
    }

    private void validateBalance(Account fromAccount, double amount) {

        if (fromAccount.getBalance() < amount) {
            throw new InsufficientBalanceException(
                    "Insufficient balance for transfer"
            );
        }
    }
}
```

// TransactionResult

```java
import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public class TransactionResult {

    private boolean success;
    private String message;
    private Long fromAccountId;
    private Long toAccountId;
    private double amount;
}
```

// Custom Exceptions

```java
public class AccountNotFoundException extends RuntimeException {

    public AccountNotFoundException(String message) {
        super(message);
    }
}
```

```java
public class InsufficientBalanceException extends RuntimeException {

    public InsufficientBalanceException(String message) {
        super(message);
    }
}
```

```java
public class InvalidTransferException extends RuntimeException {

    public InvalidTransferException(String message) {
        super(message);
    }
}
```

// JUnit 5 Test Case - Chuyển tiền thất bại do không đủ số dư

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
public class AccountServiceTest {

    @Mock
    private AccountRepository accountRepository;

    @InjectMocks
    private AccountService accountService;

    @Test
    void shouldThrowExceptionWhenBalanceIsNotEnough() {

        Long fromAccountId = 1L;
        Long toAccountId = 2L;
        double amount = 1_000_000;

        Account fromAccount = new Account();
        fromAccount.setId(fromAccountId);
        fromAccount.setBalance(500_000);

        Account toAccount = new Account();
        toAccount.setId(toAccountId);
        toAccount.setBalance(200_000);

        when(accountRepository.findById(fromAccountId))
                .thenReturn(Optional.of(fromAccount));

        when(accountRepository.findById(toAccountId))
                .thenReturn(Optional.of(toAccount));

        assertThrows(
                InsufficientBalanceException.class,
                () -> accountService.transfer(
                        fromAccountId,
                        toAccountId,
                        amount
                )
        );

        verify(accountRepository, never()).save(fromAccount);
        verify(accountRepository, never()).save(toAccount);
    }
}
```

// Nhận xét cá nhân

Qua bài này, em thấy một đoạn code “chạy được” chưa chắc đã an toàn trong dự án thực tế. Với nghiệp vụ chuyển tiền, nếu thiếu validation, transaction và logging thì rủi ro rất lớn.

Quy trình cải tiến 3 vòng giúp AI nâng cấp code theo từng lớp: đầu tiên là chống lỗi runtime và dữ liệu sai, sau đó cải thiện khả năng bảo trì bằng Spring Boot, Lombok, logging và transaction, cuối cùng bổ sung kết quả trả về cùng test case để kiểm chứng nghiệp vụ. Đây là cách dùng AI hiệu quả hơn so với việc chỉ yêu cầu “sửa code” một lần.
