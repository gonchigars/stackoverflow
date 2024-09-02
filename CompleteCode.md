### Project Setup

Create a new Spring Boot project using [Spring Initializr](https://start.spring.io/) with the following dependencies:
- Spring Web
- Spring Data JPA
- H2 Database
- Spring Security
- Lombok

### Directory Structure
```
src
└── main
    └── java
        └── com
            └── example
                └── stackoverflow
                    ├── controller
                    ├── model
                    ├── repository
                    ├── service
                    └── config
└── resources
    ├── application.properties
```

### 1. User Entity

```java
package com.example.stackoverflow.model;

import lombok.Data;

import javax.persistence.*;
import java.time.LocalDateTime;

@Data
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String password;

    private int reputation = 0;

    private LocalDateTime createdAt = LocalDateTime.now();
}
```

### 2. Question Entity

```java
package com.example.stackoverflow.model;

import lombok.Data;

import javax.persistence.*;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Data
@Entity
@Table(name = "questions")
public class Question {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(nullable = false, length = 5000)
    private String body;

    @ManyToOne
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    private int voteCount = 0;

    private LocalDateTime createdAt = LocalDateTime.now();

    @OneToMany(mappedBy = "question", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Answer> answers = new ArrayList<>();
}
```

### 3. Answer Entity

```java
package com.example.stackoverflow.model;

import lombok.Data;

import javax.persistence.*;
import java.time.LocalDateTime;

@Data
@Entity
@Table(name = "answers")
public class Answer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 5000)
    private String body;

    @ManyToOne
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne
    @JoinColumn(name = "question_id", nullable = false)
    private Question question;

    private int voteCount = 0;

    private LocalDateTime createdAt = LocalDateTime.now();
}
```

### 4. Vote Entity

```java
package com.example.stackoverflow.model;

import lombok.Data;

import javax.persistence.*;

@Data
@Entity
@Table(name = "votes")
public class Vote {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne
    @JoinColumn(name = "question_id")
    private Question question;

    @ManyToOne
    @JoinColumn(name = "answer_id")
    private Answer answer;

    private int value; // 1 for upvote, -1 for downvote
}
```

### 5. Repositories

**UserRepository.java**
```java
package com.example.stackoverflow.repository;

import com.example.stackoverflow.model.User;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
}
```

**QuestionRepository.java**
```java
package com.example.stackoverflow.repository;

import com.example.stackoverflow.model.Question;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface QuestionRepository extends JpaRepository<Question, Long> {
    @Query("SELECT q FROM Question q WHERE LOWER(q.title) LIKE LOWER(CONCAT('%', :keyword, '%')) OR LOWER(q.body) LIKE LOWER(CONCAT('%', :keyword, '%'))")
    List<Question> searchQuestions(String keyword);
}
```

**AnswerRepository.java**
```java
package com.example.stackoverflow.repository;

import com.example.stackoverflow.model.Answer;
import org.springframework.data.jpa.repository.JpaRepository;

public interface AnswerRepository extends JpaRepository<Answer, Long> {
}
```

**VoteRepository.java**
```java
package com.example.stackoverflow.repository;

import com.example.stackoverflow.model.Vote;
import org.springframework.data.jpa.repository.JpaRepository;

public interface VoteRepository extends JpaRepository<Vote, Long> {
}
```

### 6. Services

**UserService.java**
```java
package com.example.stackoverflow.service;

import com.example.stackoverflow.model.User;
import com.example.stackoverflow.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    public User registerUser(User user) {
        user.setPassword(passwordEncoder.encode(user.getPassword()));
        return userRepository.save(user);
    }

    public User getUserByUsername(String username) {
        return userRepository.findByUsername(username)
                .orElseThrow(() -> new RuntimeException("User not found"));
    }
}
```

**QuestionService.java**
```java
package com.example.stackoverflow.service;

import com.example.stackoverflow.model.Question;
import com.example.stackoverflow.repository.QuestionRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class QuestionService {

    @Autowired
    private QuestionRepository questionRepository;

    public Question askQuestion(Question question) {
        return questionRepository.save(question);
    }

    public List<Question> getAllQuestions() {
        return questionRepository.findAll();
    }

    public Question getQuestionById(Long id) {
        return questionRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Question not found"));
    }

    public List<Question> searchQuestions(String keyword) {
        return questionRepository.searchQuestions(keyword);
    }
}
```

**AnswerService.java**
```java
package com.example.stackoverflow.service;

import com.example.stackoverflow.model.Answer;
import com.example.stackoverflow.repository.AnswerRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class AnswerService {

    @Autowired
    private AnswerRepository answerRepository;

    public Answer postAnswer(Answer answer) {
        return answerRepository.save(answer);
    }
}
```

**VoteService.java**
```java
package com.example.stackoverflow.service;

import com.example.stackoverflow.model.Vote;
import com.example.stackoverflow.repository.VoteRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class VoteService {

    @Autowired
    private VoteRepository voteRepository;

    public Vote castVote(Vote vote) {
        return voteRepository.save(vote);
    }
}
```

### 7. Controllers

**UserController.java**
```java
package com.example.stackoverflow.controller;

import com.example.stackoverflow.model.User;
import com.example.stackoverflow.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping("/register")
    public ResponseEntity<User> registerUser(@RequestBody User user) {
        return ResponseEntity.ok(userService.registerUser(user));
    }
}
```

**QuestionController.java**
```java
package com.example.stackoverflow.controller;

import com.example.stackoverflow.model.Question;
import com.example.stackoverflow.service.QuestionService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/questions")
public class QuestionController {

    @Autowired
    private QuestionService questionService;

    @PostMapping
    public ResponseEntity<Question> askQuestion(@RequestBody Question question) {
        return ResponseEntity.ok(questionService.askQuestion(question));
    }

    @GetMapping
    public ResponseEntity<List<Question>> getAllQuestions() {
        return ResponseEntity.ok(questionService.getAllQuestions());
    }

    @GetMapping("/{id}")
    public ResponseEntity<Question> getQuestionById(@PathVariable Long id) {
        return ResponseEntity.ok(questionService.getQuestionById(id));
    }

    @GetMapping("/search")
    public ResponseEntity<List<Question>> searchQuestions(@RequestParam String keyword) {
        return ResponseEntity.ok(questionService.searchQuestions(keyword));
    }
}
```

**AnswerController.java**
```java
package com.example.stackoverflow.controller;

import com.example.stackoverflow.model.Answer;
import com.example.stackoverflow.service.AnswerService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/answers")
public class AnswerController {

    @Autowired
    private AnswerService answerService;

    @PostMapping
    public ResponseEntity<Answer> postAnswer(@RequestBody Answer answer) {
        return ResponseEntity.ok(answerService.postAnswer(answer));
    }
}
```

**VoteController.java**
```java


package com.example.stackoverflow.controller;

import com.example.stackoverflow.model.Vote;
import com.example.stackoverflow.service.VoteService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/votes")
public class VoteController {

    @Autowired
    private VoteService voteService;

    @PostMapping
    public ResponseEntity<Vote> castVote(@RequestBody Vote vote) {
        return ResponseEntity.ok(voteService.castVote(vote));
    }
}
```

### 8. Security Configuration

**SecurityConfig.java**
```java
package com.example.stackoverflow.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeRequests()
                .antMatchers("/api/users/register").permitAll()
                .anyRequest().authenticated()
            .and()
            .httpBasic();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 9. Application Properties

**application.properties**
```properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.h2.console.enabled=true
```

### 10. Running the Application

Run the application using your IDE or by executing the following command in your terminal:

```bash
mvn spring-boot:run
```

### Conclusion

This implementation provides the complete code for building a simplified version of Stack Overflow using Spring Boot. The tutorial covered creating the necessary entities, services, repositories, controllers, and security configuration. The project can be extended further by adding more features such as comment functionality, tags management, and more sophisticated search capabilities.
