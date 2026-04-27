# Лабораторная работа №5
## Homework
### Тестирование библиотеки Banking

[![CI](https://github.com/Mimocake/TIMP-banking/actions/workflows/CI.yml/badge.svg(https://github.com/Mimocake/TIMP-banking/actions/workflows/CI.yml)
[![Coverage Status](https://coveralls.io/repos/github/Mimocake/TIMP-banking/badge.svg?branch=main)](https://coveralls.io/github/Mimocake/TIMP-banking?branch=main)

После клонирования исходных файлов, а также подключения библиотеки **GTest**, для начала, напишем необходимые `CMakeLists.txt` для сборки проекта. Рассмотрим `CMakeLists.txt` внутри папки `banking`:
```cmake
cmake_minimum_required(VERSION 3.10)
project(banking)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if (NOT TARGET account)
	add_library(account STATIC ${CMAKE_CURRENT_SOURCE_DIR}/Account.cpp)
endif()

if (NOT TARGET transaction)
	add_library(transaction STATIC ${CMAKE_CURRENT_SOURCE_DIR}/Transaction.cpp)
endif()

target_include_directories(account PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_include_directories(transaction PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_link_directories(transaction PUBLIC account)

if (COLLECT_COVERAGE)
	target_compile_options(account PRIVATE -O0 -g --coverage)
	target_link_options(account PRIVATE --coverage)
	target_compile_options(transaction PRIVATE -O0 -g --coverage)
	target_link_options(transaction PRIVATE --coverage)
endif()
```
Здесь, мы собираем 2 библиотечки для каждого из файлов, а также опционально задаем флаги компилятору для сбора данных о покрытии.

И теперь посмотрим на главный `CMakeLists.txt`:
```cmake
cmake_minimum_required(VERSION 3.10)
project(banking_tests)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TESTS "Build tests" OFF)
option(COLLECT_COVERAGE "Collect coverage" OFF)

add_subdirectory(banking)

if (BUILD_TESTS)
	enable_testing()
	add_subdirectory(third-party/gtest)
	
	file(GLOB TEST_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/tests/*.cpp")
	add_executable(check ${TEST_SOURCES})
	target_link_libraries(check account transaction gtest_main gmock_main)

	if (COLLECT_COVERAGE)
		target_link_libraries(check gcov)
		target_compile_options(check PRIVATE -O0 -g --coverage)
		target_link_options(check PRIVATE --coverage)
	endif()

	add_test(NAME check COMMAND check)
endif()
```
Здесь мы подключаем субдиректорию, а потом опционально собираем тесты, которые будут находиться в папке `tests`, и еще опционально собираем данные о покрытии (так же добавив флаги компилятору).

Теперь перейдем к самой папке `tests`.

Тут будет 2 файла: один будет тестировать `Account`, а другой - `Transaction`. 

Для начала рассмотрим тестировку `Account`:
```cpp
#include <Account.h>
#include <gtest/gtest.h>

class AccountFixture : public testing::Test {
public:
	Account* acc;
	void SetUp() { acc = new Account(123, 1000); }
	void TearDown() { delete acc; }
};

TEST_F(AccountFixture, GetBalance) {
	EXPECT_EQ(acc->GetBalance(), 1000);
}

TEST_F(AccountFixture, ChangeBalanceGood) {
	acc->Lock();
	acc->ChangeBalance(200);
	EXPECT_EQ(acc->GetBalance(), 1200);
}

TEST_F(AccountFixture, ChangeBalanceBad) {
	EXPECT_THROW(acc->ChangeBalance(100), std::runtime_error);
}

TEST_F(AccountFixture, GetID) {
	EXPECT_EQ(acc->id(), 123);
}

TEST_F(AccountFixture, LockTwice) {
	acc->Lock();
	EXPECT_THROW(acc->Lock(), std::runtime_error);
}

TEST_F(AccountFixture, LockAndUnlock) {
	acc->Lock();
	acc->Unlock();
	EXPECT_NO_THROW(acc->Lock());
}
```
Здесь мы создаем так называемую фикстуру класса `Account`. Это нужно для того, чтобы при каждом запуске теста у нас уже был instance объекта, который не нужно задавать каждый раз. И также такой способ автоматически покрывает деструктор. В самих тестах мы делаем корректные и не очень вещи с этим объектом, чтобы покрыть все возможные варианты поведения.

Теперь посмотрим на тестирование класса `Transaction`.
```cpp
#include <Account.h>
#include <Transaction.h>
#include <gtest/gtest.h>
#include <gmock/gmock.h>

class AccountMock : public Account {
public:
	AccountMock(int id, int balance) : Account(id, balance) {}
	
	MOCK_METHOD(int, GetBalance, (), (const, override));
  	MOCK_METHOD(void, ChangeBalance, (int diff), (override));
  	MOCK_METHOD(void, Lock, (), (override));
  	MOCK_METHOD(void, Unlock, (), (override));
};

class TransactionFixture : public testing::Test {
public:
	Transaction* tr;
	AccountMock* from;
	AccountMock* to;
	void SetUp () override { 
		tr = new Transaction;
		from = new testing::NiceMock<AccountMock>(1, 1000);
		to = new testing::NiceMock<AccountMock>(2, 1000);
	}
	void TearDown () override { 
		delete tr;
		delete from;
		delete to;
	}
};

TEST_F(TransactionFixture, Fee) {
	EXPECT_EQ(tr->fee(), 1);
}

TEST_F(TransactionFixture, SetFee) {
	tr->set_fee(10);
	EXPECT_EQ(tr->fee(), 10);
}

TEST_F(TransactionFixture, SuccessfulTransfer) {
	EXPECT_CALL(*to, ChangeBalance(200)).Times(1);
	EXPECT_CALL(*from, GetBalance()).WillOnce(testing::Return(1000)).WillRepeatedly(testing::Return(799));
	EXPECT_CALL(*to, GetBalance()).WillRepeatedly(testing::Return(1200));

	EXPECT_TRUE(tr->Make(*from, *to, 200));
}

TEST_F(TransactionFixture, TransferToYourself) {
	EXPECT_THROW(tr->Make(*from, *from, 200), std::logic_error);
}

TEST_F(TransactionFixture, NegativeSumTransfer) {
	EXPECT_THROW(tr->Make(*from, *to, -100), std::invalid_argument);
}

TEST_F(TransactionFixture, TooSmallSumTransfer) {
	EXPECT_THROW(tr->Make(*from, *to, 70), std::logic_error);
}

TEST_F(TransactionFixture, TooBigFee) {
	tr->set_fee(60);
	EXPECT_FALSE(tr->Make(*from, *to, 100));
}

TEST_F(TransactionFixture, TooBigSumTransfer) {
	EXPECT_CALL(*from, GetBalance()).WillRepeatedly(testing::Return(1000));
	EXPECT_CALL(*to, ChangeBalance(1200)).Times(1);
	EXPECT_CALL(*to, ChangeBalance(-1200)).Times(1);
	EXPECT_CALL(*to, GetBalance()).WillRepeatedly(testing::Return(1000));
	EXPECT_FALSE(tr->Make(*from, *to, 1200));
}
```
Здесь дела обстоят уже посложнее. Объекты класса `Transaction` используют объекты класса `Account`, то есть существует зависимость между двумя этим классами. Соответсвенно и тестирование придется проводить в виде матрешки, что не очень удобно. Но для этих целей существуют Mock-объекты. С их помощью мы можем реально не тестировать внутренние зависимсоти, а просто их симулировать, и концетрироваться только на "внешнем" классе. Поэтому создадим класс `AccountMock`, унаследованный от `Account`, а все его методы создадим с помощью макроса из gtest, кроме метода `id`, который не объявлен виртуальным, и с ним такой фокус не пройдет. Дальше создадим уже знакомую фикстуру для `Transaction` и добавим туда два мока. Дальше при тестировке мы будем работать не с настоящими объектами аккаунтами, а только с их моками. Но поскольку это фиктивные объекты, реально они ничего полезного не делают, и мы должны сами говорить, что они сделают/возвратят при потенциальном вызове. Это поведение описано в макросе EXPECT_CALL. То есть мы искусственно делаем класс `Account` полностью корректным, сами симулируя его поведение, а тестируем только оболочку `Transaction`.

Давайте проверим, что все работает.

Запускаем симейк со следующими флагами
```sh
cmake -H. -B build -D BUILD_TESTS=ON -D COLLECT_COVERAGE=ON
```
Которые вызовут сборку тестов и информации о покрытии. 

<details>

<summary> Вывод команды </summary>

```
-- The C compiler identification is GNU 11.4.0
-- The CXX compiler identification is GNU 11.4.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Looking for pthread.h
-- Looking for pthread.h - found
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Success
-- Found Threads: TRUE  
-- Configuring done
-- Generating done
-- Build files have been written to: /home/univer/Desktop/timp/Mimocake/workspace/projects/bankinggg/build
```

</details>

Дальше собираем сам проект
```sh
cmake --build build
```

<details>

<summary> Вывод </summary>

```
[  6%] Building CXX object third-party/gtest/googletest/CMakeFiles/gtest.dir/src/gtest-all.cc.o
[ 13%] Linking CXX static library ../../../lib/libgtest.a
[ 13%] Built target gtest
[ 20%] Building CXX object third-party/gtest/googletest/CMakeFiles/gtest_main.dir/src/gtest_main.cc.o
[ 26%] Linking CXX static library ../../../lib/libgtest_main.a
[ 26%] Built target gtest_main
[ 33%] Building CXX object banking/CMakeFiles/account.dir/Account.cpp.o
[ 40%] Linking CXX static library libaccount.a
[ 40%] Built target account
[ 46%] Building CXX object banking/CMakeFiles/transaction.dir/Transaction.cpp.o
[ 53%] Linking CXX static library libtransaction.a
[ 53%] Built target transaction
[ 60%] Building CXX object third-party/gtest/googlemock/CMakeFiles/gmock.dir/src/gmock-all.cc.o
[ 66%] Linking CXX static library ../../../lib/libgmock.a
[ 66%] Built target gmock
[ 73%] Building CXX object third-party/gtest/googlemock/CMakeFiles/gmock_main.dir/src/gmock_main.cc.o
[ 80%] Linking CXX static library ../../../lib/libgmock_main.a
[ 80%] Built target gmock_main
[ 86%] Building CXX object CMakeFiles/check.dir/tests/AccountTest.cpp.o
[ 93%] Building CXX object CMakeFiles/check.dir/tests/TransactionTest.cpp.o
[100%] Linking CXX executable check
[100%] Built target check
```

</details>

И запускаем тесты
```sh
build/check
```

```
[==========] Running 14 tests from 2 test suites.
[----------] Global test environment set-up.
[----------] 6 tests from AccountFixture
[ RUN      ] AccountFixture.GetBalance
[       OK ] AccountFixture.GetBalance (0 ms)
[ RUN      ] AccountFixture.ChangeBalanceGood
[       OK ] AccountFixture.ChangeBalanceGood (0 ms)
[ RUN      ] AccountFixture.ChangeBalanceBad
[       OK ] AccountFixture.ChangeBalanceBad (0 ms)
[ RUN      ] AccountFixture.GetID
[       OK ] AccountFixture.GetID (0 ms)
[ RUN      ] AccountFixture.LockTwice
[       OK ] AccountFixture.LockTwice (0 ms)
[ RUN      ] AccountFixture.LockAndUnlock
[       OK ] AccountFixture.LockAndUnlock (0 ms)
[----------] 6 tests from AccountFixture (0 ms total)

[----------] 8 tests from TransactionFixture
[ RUN      ] TransactionFixture.Fee
[       OK ] TransactionFixture.Fee (0 ms)
[ RUN      ] TransactionFixture.SetFee
[       OK ] TransactionFixture.SetFee (0 ms)
[ RUN      ] TransactionFixture.SuccessfulTransfer
1 send to 2 $200
Balance 1 is 799
Balance 2 is 1200
[       OK ] TransactionFixture.SuccessfulTransfer (0 ms)
[ RUN      ] TransactionFixture.TransferToYourself
[       OK ] TransactionFixture.TransferToYourself (0 ms)
[ RUN      ] TransactionFixture.NegativeSumTransfer
[       OK ] TransactionFixture.NegativeSumTransfer (0 ms)
[ RUN      ] TransactionFixture.TooSmallSumTransfer
[       OK ] TransactionFixture.TooSmallSumTransfer (0 ms)
[ RUN      ] TransactionFixture.TooBigFee
[       OK ] TransactionFixture.TooBigFee (0 ms)
[ RUN      ] TransactionFixture.TooBigSumTransfer
1 send to 2 $1200
Balance 1 is 1000
Balance 2 is 1000
[       OK ] TransactionFixture.TooBigSumTransfer (0 ms)
[----------] 8 tests from TransactionFixture (0 ms total)

[----------] Global test environment tear-down
[==========] 14 tests from 2 test suites ran. (1 ms total)
[  PASSED  ] 14 tests.
```

Как видим, все тесты прошли успешно.

Теперь можно посмотреть на покрытие кода. Запустим команду:
```sh
lcov --capture --directory build/banking --output-file=cov.info
```

<details>

<summary> Вывод </summary>

```
Capturing coverage data from build/banking
Subroutine read_intermediate_text redefined at /usr/bin/geninfo line 2623.
Subroutine read_intermediate_json redefined at /usr/bin/geninfo line 2655.
Subroutine intermediate_text_to_info redefined at /usr/bin/geninfo line 2703.
Subroutine intermediate_json_to_info redefined at /usr/bin/geninfo line 2792.
Subroutine get_output_fd redefined at /usr/bin/geninfo line 2872.
Subroutine print_gcov_warnings redefined at /usr/bin/geninfo line 2900.
Subroutine process_intermediate redefined at /usr/bin/geninfo line 2930.
Found gcov version: 11.4.0
Using intermediate gcov format
Scanning build/banking for .gcda files ...
Found 2 data files in build/banking
Processing account.dir/Account.cpp.gcda
Processing transaction.dir/Transaction.cpp.gcda
Finished .info-file creation
```

</details>

Которая соберет ранее созданные компилятом данные о покрытии в один файл `cov.info`. Можно просто вывести его содержание, но это не особо удобно, лучше воспользоваться еще одной командой:
```sh
lcov --list cov.info
```
Которая выведет все в более очевидном виде:
```
Reading tracefile cov.info
                      |Lines       |Functions  |Branches    
Filename              |Rate     Num|Rate    Num|Rate     Num
============================================================
[/home/univer/Desktop/timp/Mimocake/workspace/projects/bankinggg/banking/]
Account.cpp           | 100%     16| 100%     8|    -      0
Transaction.cpp       | 100%     33| 100%     9|    -      0
============================================================
                Total:| 100%     49| 100%    17|    -      0
```

Как видно, покрытие составляет 100%, а значит можно задуматься о непрерывной интеграции.

### Непрерывная интеграция

Создадим скрипт для **Github Actions**:
```yaml
name: linux
on: push
jobs:
  build_and_test:
    runs-on: ubuntu-latest
    steps:
      - name: Set GCC 11
        uses: egor-tensin/setup-gcc@v1
        with:
          version: 11
          platform: x64
      - name: Get repo files
        uses: actions/checkout@v6
        with:
          submodules: true
      - name: Configure build directory and options
        run: cmake -H. -B build -D BUILD_TESTS=ON -D COLLECT_COVERAGE=ON
      - name: Build project
        run: cmake --build build
      - name: Run tests
        run: ./build/check
      - name: Install lcov
        run: sudo apt install lcov
      - name: Make coverage configuration file
        run: lcov --capture --directory build/banking --gcov-tool /usr/bin/gcov-11 --output-file=cov.info
      - name: Get coverage percentage
        run: lcov --list cov.info
      - name: Coveralls Upload
        uses: coverallsapp/github-action@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          file: cov.info
```

Тут мы на всякий случай устанавливаем GCC 11 версии, так как у меня на компьютере стоит именно эта версия, и выполняем все команды, описанные выше. И на последок выполяем интеграцию с **Coveralls**. К сожалению, почему-то Github Actions выдает какие-то неправильные данные о покрытии, но Coveralls работает получше и выдает честные 100%.
