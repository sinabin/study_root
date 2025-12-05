# HTTP Status Code 적용 리팩토링 기획서

## 1. 개요

### 1.1 현재 문제점
- 모든 API 응답이 HTTP Status 200 OK로 반환됨
- 에러 구분을 응답 body의 `msgCd` 필드로만 처리
- RESTful API 원칙 미준수
- 표준 HTTP 모니터링/로깅 도구 활용 불가

### 1.2 개선 목표
- HTTP Status Code를 의미론적으로 올바르게 사용
- Spring의 `ResponseEntity`를 활용한 표준화된 응답 처리
- 기존 `DtoAssembler` 구조 유지하면서 HTTP Status 추가
- 하위 호환성 보장 (클라이언트 영향 최소화)

---

## 2. 아키텍처 설계

### 2.1 레이어 구조 변경

```
[Before]
Controller (extends BaseService)
  → DtoAssembler 반환
  → 항상 200 OK

[After]
Controller (extends BaseController)
  → ResponseEntity<DtoAssembler> 반환
  → 적절한 HTTP Status Code
```

### 2.2 주요 변경 컴포넌트

| 컴포넌트 | 변경 내용 | 비고 |
|---------|----------|------|
| `BaseService` | 이름 변경: `BaseController` | 역할에 맞게 네이밍 변경 |
| `DtoAssembler` | 그대로 유지 | 응답 데이터 구조 유지 |
| `GeneralExceptionHandler` | HTTP Status 반영 | 에러별 적절한 상태 코드 |
| `CmmJsonResponse` | HTTP Status 파라미터 추가 | 기존 로직 유지하며 확장 |
| 각 Controller | `ResponseEntity` 사용 | 단계적 마이그레이션 |

---

## 3. 구현 상세

### 3.1 HTTP Status Code 매핑 전략

#### 3.1.1 기본 매핑 규칙

| msgCd | 현재 의미 | HTTP Status | 비고 |
|-------|----------|-------------|------|
| 000 | 정상 (데이터 있음) | 200 OK | 조회 성공 |
| 010 | 정상 (데이터 없음) | 200 OK | 조회 성공이나 빈 결과 |
| 100 | 저장/수정/삭제 성공 | 200 OK (수정), 201 Created (생성) | 메서드에 따라 구분 |
| 110 | 에러 발생 | 400/404/500 등 | 에러 종류에 따라 구분 |

#### 3.1.2 상세 매핑

```
조회 (SELECT)
├─ 성공 (데이터 있음): 200 OK
├─ 성공 (데이터 없음): 200 OK (body에 빈 배열)
└─ 실패: 400 Bad Request / 404 Not Found

생성 (INSERT)
├─ 성공: 201 Created
├─ 실패 (유효성 검증): 400 Bad Request
└─ 실패 (중복): 409 Conflict

수정 (UPDATE)
├─ 성공: 200 OK
├─ 실패 (대상 없음): 404 Not Found
└─ 실패 (유효성 검증): 400 Bad Request

삭제 (DELETE)
├─ 성공: 200 OK (또는 204 No Content)
├─ 실패 (대상 없음): 404 Not Found
└─ 실패 (제약조건): 409 Conflict

비즈니스 로직 에러
├─ 유효성 검증 실패: 400 Bad Request
├─ 인증 실패: 401 Unauthorized
├─ 권한 없음: 403 Forbidden
├─ 리소스 없음: 404 Not Found
└─ 서버 에러: 500 Internal Server Error
```

---

### 3.2 BaseController 리팩토링

#### 3.2.1 기존 BaseService → BaseController로 변경

**파일: `com/fw/cmm/dataSet/BaseController.java`**

```java
package com.fw.cmm.dataSet;

import java.util.List;
import java.util.Map;
import java.util.Map.Entry;

import org.apache.commons.collections.MapUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.MessageSource;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.lang.Nullable;

import com.fw.cmm.utils.CmmHashMap;

/**
 * 메뉴경로 : 공통
 * 프로그램개요 : 화면으로 가는 데이터를 정규화하여 처리하기 위한 Base Controller
 *                HTTP Status Code를 적절하게 반환
 * 변경이력
 * ------------------------------------------------------------
 * 버전      개발일자       이름      설명(수정내용)
 * ------------------------------------------------------------
 * 1.0       2024. 6. 5.    com       최초 개발
 * 2.0       2025. 11. 11.  refactor  HTTP Status Code 적용
 *
 * @author com
 * @since  2024. 6. 5.
 */
public class BaseController {

    public BaseController() {}

    @Autowired
    protected MessageSource messageSource;

    @Autowired
    protected CustomMessage customMessage;

    @Autowired
    protected DtoAssembler assembler;

    /**
     * <p>단일 데이터 리스트만 넘길때 사용
     * 자동으로 메세지 셋팅
     * 데이터 : "list"
     * 메시지 : "message"</p>
     * @param result 조회 결과 리스트
     * @return ResponseEntity<DtoAssembler> (200 OK)
     * @author  com
     * @date    2024. 6. 26.
     */
    protected ResponseEntity<DtoAssembler> afterSelectList(List<CmmHashMap<String, Object>> result) {
        assembler = new DtoAssembler();

        if (result != null && result.size() > 0) {
            customMessage.setMsgCd("000");
            customMessage.setDetailMsg(messageSource.getMessage("fw.cmm.023", new String[]{"조회"}, null));
        } else {
            customMessage.setMsgCd("010");
            customMessage.setDetailMsg(messageSource.getMessage("fw.cmm.017", new String[]{"데이터"}, null));
        }

        assembler.put("list", result);
        assembler.putMessage(customMessage);

        return ResponseEntity.ok(assembler);
    }

    /**
     * <p>맵형태의 데이터 넘길때 사용
     * 맵데이터에서 개발자가 따로 CustomMessage를 셋팅하지 않으면 자동 추가</p>
     * @param map 응답 데이터 맵
     * @return ResponseEntity<DtoAssembler> (200 OK)
     * @author  com
     * @date    2024. 6. 26.
     */
    protected ResponseEntity<DtoAssembler> afterCommonData(Map<String, Object> map) {
        assembler = new DtoAssembler();
        customMessage = (CustomMessage) MapUtils.getObject(map, "message");

        if (null == customMessage) {
            customMessage = new CustomMessage();
            customMessage.setMsgCd("000");
            customMessage.setDetailMsg(messageSource.getMessage("fw.cmm.031", new String[]{""}, null));
            assembler.putMessage(customMessage);
        }

        if (map != null && !map.isEmpty()) {
            for (Entry<String, Object> entry : map.entrySet()) {
                assembler.put(entry.getKey(), entry.getValue());
            }
        }

        return ResponseEntity.ok(assembler);
    }

    /**
     * <p>INSERT 성공시 리턴</p>
     * @param cnt 성공 갯수
     * @return ResponseEntity<DtoAssembler> (201 Created)
     * @author  com
     * @date    2025. 11. 11.
     */
    @Nullable
    protected ResponseEntity<DtoAssembler> afterInsert(@Nullable Integer cnt) {
        assembler = new DtoAssembler();
        String strMessage = cnt + "건.\n" + messageSource.getMessage("fw.cmm.001", new String[]{}, null);

        customMessage.setMsgCd("100");
        customMessage.setDetailMsg(strMessage);
        assembler.putMessage(customMessage);

        return ResponseEntity.status(HttpStatus.CREATED).body(assembler);
    }

    /**
     * <p>UPDATE 성공시 리턴</p>
     * @param cnt 성공 갯수
     * @return ResponseEntity<DtoAssembler> (200 OK)
     * @author  com
     * @date    2025. 11. 11.
     */
    @Nullable
    protected ResponseEntity<DtoAssembler> afterUpdate(@Nullable Integer cnt) {
        assembler = new DtoAssembler();
        String strMessage = cnt + "건.\n" + messageSource.getMessage("fw.cmm.003", new String[]{}, null);

        customMessage.setMsgCd("100");
        customMessage.setDetailMsg(strMessage);
        assembler.putMessage(customMessage);

        return ResponseEntity.ok(assembler);
    }

    /**
     * <p>DELETE 성공시 리턴</p>
     * @param cnt 성공 갯수
     * @return ResponseEntity<DtoAssembler> (200 OK)
     * @author  com
     * @date    2025. 11. 11.
     */
    @Nullable
    protected ResponseEntity<DtoAssembler> afterDelete(@Nullable Integer cnt) {
        assembler = new DtoAssembler();
        String strMessage = cnt + "건.\n" + messageSource.getMessage("fw.cmm.005", new String[]{}, null);

        customMessage.setMsgCd("100");
        customMessage.setDetailMsg(strMessage);
        assembler.putMessage(customMessage);

        return ResponseEntity.ok(assembler);
    }

    /**
     * <p>범용 저장 메서드 (기존 호환성 유지용 - Deprecated)</p>
     * @deprecated 대신 afterInsert, afterUpdate, afterDelete 사용 권장
     */
    @Deprecated
    @Nullable
    protected ResponseEntity<DtoAssembler> saveData(String strMethod, @Nullable Integer cnt) {
        if ("insert".equals(strMethod)) {
            return afterInsert(cnt);
        } else if ("update".equals(strMethod)) {
            return afterUpdate(cnt);
        } else if ("delete".equals(strMethod)) {
            return afterDelete(cnt);
        }
        return ResponseEntity.ok(new DtoAssembler());
    }

    @Deprecated
    protected ResponseEntity<DtoAssembler> saveData(String strMethod) {
        return saveData(strMethod, null);
    }
}
```

---

### 3.3 Exception Handler 개선

#### 3.3.1 CustomException 확장

**파일: `com/fw/cmm/exception/CustomException.java`**

```java
package com.fw.cmm.exception;

import lombok.Data;
import lombok.EqualsAndHashCode;
import org.springframework.http.HttpStatus;

/**
 * 메뉴경로 : 공통 - Exception
 * 프로그램개요 : 단위 프로그램에서 Exception 처리를 따로 하기 위한 클래스
 * 변경이력
 * ------------------------------------------------------------
 * 버전      개발일자       이름      설명(수정내용)
 * ------------------------------------------------------------
 * 1.0       2024. 7. 18.   com       최초 개발
 * 2.0       2025. 11. 11.  refactor  HTTP Status 추가
 *
 * @author  com
 * @since   2024. 7. 18.
 */
@Data
@EqualsAndHashCode(callSuper = true)
public class CustomException extends RuntimeException {

    private static final long serialVersionUID = 1L;

    private String code;
    private Object obj;
    private String retUrl;
    private HttpStatus httpStatus;  // 추가: HTTP 상태 코드

    /**
     * 기존 생성자 (하위 호환성 유지 - 기본 400 Bad Request)
     */
    public CustomException(String strCode, Object[] arg, String strRetUrl) {
        super();
        this.code = strCode;
        this.obj = arg;
        this.retUrl = strRetUrl;
        this.httpStatus = HttpStatus.BAD_REQUEST;  // 기본값
    }

    /**
     * HTTP Status를 명시적으로 지정하는 생성자
     */
    public CustomException(String strCode, Object[] arg, String strRetUrl, HttpStatus httpStatus) {
        super();
        this.code = strCode;
        this.obj = arg;
        this.retUrl = strRetUrl;
        this.httpStatus = httpStatus;
    }

    /**
     * 편의 생성자들
     */
    public static CustomException badRequest(String code, Object[] arg) {
        return new CustomException(code, arg, "NULL", HttpStatus.BAD_REQUEST);
    }

    public static CustomException notFound(String code, Object[] arg) {
        return new CustomException(code, arg, "NULL", HttpStatus.NOT_FOUND);
    }

    public static CustomException conflict(String code, Object[] arg) {
        return new CustomException(code, arg, "NULL", HttpStatus.CONFLICT);
    }

    public static CustomException unauthorized(String code, Object[] arg) {
        return new CustomException(code, arg, "NULL", HttpStatus.UNAUTHORIZED);
    }

    public static CustomException forbidden(String code, Object[] arg) {
        return new CustomException(code, arg, "NULL", HttpStatus.FORBIDDEN);
    }
}
```

#### 3.3.2 비즈니스 예외 클래스 추가

**파일: `com/fw/cmm/exception/NotFoundException.java`**

```java
package com.fw.cmm.exception;

import org.springframework.http.HttpStatus;

/**
 * 리소스를 찾을 수 없을 때 사용하는 예외
 * HTTP 404 Not Found를 반환
 */
public class NotFoundException extends CustomException {

    public NotFoundException(String code, Object[] arg) {
        super(code, arg, "NULL", HttpStatus.NOT_FOUND);
    }

    public NotFoundException(String message) {
        super("fw.cmm.040", new String[]{message}, "NULL", HttpStatus.NOT_FOUND);
    }
}
```

**파일: `com/fw/cmm/exception/BadRequestException.java`**

```java
package com.fw.cmm.exception;

import org.springframework.http.HttpStatus;

/**
 * 잘못된 요청일 때 사용하는 예외
 * HTTP 400 Bad Request를 반환
 */
public class BadRequestException extends CustomException {

    public BadRequestException(String code, Object[] arg) {
        super(code, arg, "NULL", HttpStatus.BAD_REQUEST);
    }

    public BadRequestException(String message) {
        super("fw.cmm.036", new String[]{message}, "NULL", HttpStatus.BAD_REQUEST);
    }
}
```

**파일: `com/fw/cmm/exception/ConflictException.java`**

```java
package com.fw.cmm.exception;

import org.springframework.http.HttpStatus;

/**
 * 리소스 충돌이 발생했을 때 사용하는 예외 (예: 중복 등록)
 * HTTP 409 Conflict를 반환
 */
public class ConflictException extends CustomException {

    public ConflictException(String code, Object[] arg) {
        super(code, arg, "NULL", HttpStatus.CONFLICT);
    }

    public ConflictException(String message) {
        super("fw.cmm.041", new String[]{message}, "NULL", HttpStatus.CONFLICT);
    }
}
```

#### 3.3.3 GeneralExceptionHandler 개선

**파일: `com/fw/cmm/exception/GeneralExceptionHandler.java`**

```java
package com.fw.cmm.exception;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.util.Enumeration;
import java.util.HashMap;
import java.util.Map;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.MessageSource;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import com.fw.cmm.dataSet.CustomMessage;
import com.fw.cmm.dataSet.DtoAssembler;
import com.fw.cmm.utils.CmmJsonResponse;
import com.fw.cmm.utils.SecurityUtils;
import com.fw.cmm.utils.StringUtils;
import com.fw.config.common.CommonService;
import com.fw.config.security.userDetail.UserInfo;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;

/**
 * 메뉴경로 : 공통
 * 프로그램개요 : 에러처리 - HTTP Status Code를 적절하게 반환
 * 변경이력
 * ------------------------------------------------------------
 * 버전      개발일자       이름      설명(수정내용)
 * ------------------------------------------------------------
 * 1.0       2024. 5. 10.   com       최초 개발
 * 2.0       2025. 11. 11.  refactor  HTTP Status 추가
 *
 * @author com
 * @since  2024. 5. 10.
 */
@Slf4j
@RestControllerAdvice
public class GeneralExceptionHandler {

    @Autowired
    private MessageSource messageSource;

    @Autowired
    private CommonService commonService;

    /**
     * CustomException 예외 처리 (HTTP Status 반영)
     */
    @ExceptionHandler({CustomException.class})
    public ResponseEntity<Map<String, Object>> handleCustomException(
            HttpServletRequest req,
            HttpServletResponse res,
            CustomException ex) {

        String message = messageSource.getMessage(ex.getCode(), (Object[]) ex.getObj(), null);
        HttpStatus status = ex.getHttpStatus() != null ? ex.getHttpStatus() : HttpStatus.BAD_REQUEST;

        saveErrorLog(req, ex, message);  // 에러 로그 저장

        // AJAX 요청인 경우 JSON 응답
        boolean isAjax = "XMLHttpRequest".equals(req.getHeader("X-Requested-With"));
        if (isAjax) {
            Map<String, Object> errorResponse = new HashMap<>();
            CustomMessage customMessage = new CustomMessage();
            customMessage.setMsgCd("110");
            customMessage.setDetailMsg(message);
            customMessage.setUrl(ex.getRetUrl());

            errorResponse.put("message", customMessage);

            return ResponseEntity.status(status).body(errorResponse);
        }

        // 기존 방식 (리다이렉트) - 하위 호환성 유지
        CmmJsonResponse.returnJson(req, res, "110", message, ex.getRetUrl(), status.value());
        return null;
    }

    /**
     * 일반 Exception 처리 (500 Internal Server Error)
     */
    @ExceptionHandler({Exception.class})
    public ResponseEntity<Map<String, Object>> handleGeneralException(
            HttpServletRequest req,
            HttpServletResponse res,
            Exception ex) {

        String message = messageSource.getMessage("fw.cmm.033", new String[]{""}, null);
        saveErrorLog(req, ex, null);

        boolean isAjax = "XMLHttpRequest".equals(req.getHeader("X-Requested-With"));
        if (isAjax) {
            Map<String, Object> errorResponse = new HashMap<>();
            CustomMessage customMessage = new CustomMessage();
            customMessage.setMsgCd("110");
            customMessage.setDetailMsg(message);
            customMessage.setUrl("NULL");

            errorResponse.put("message", customMessage);

            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(errorResponse);
        }

        CmmJsonResponse.returnJson(req, res, "110", message, "NULL", HttpStatus.INTERNAL_SERVER_ERROR.value());
        return null;
    }

    /**
     * 에러 저장
     */
    private void saveErrorLog(HttpServletRequest req, Exception e, String customMessage) {
        log.debug("========== HERE!!!!! e.printStackTrace() START ==========");
        e.printStackTrace();
        log.debug("========== HERE!!!!! e.printStackTrace() END   ==========");

        Map<String, String> saveErrorLogMap = new HashMap<>();
        UserInfo userInfo = SecurityUtils.getCurrentUserInfo();

        String strErrorUri = null == req.getRequestURI() || "".equals(req.getRequestURI()) ? "NULL" : req.getRequestURI();
        String strServerInfo[] = StringUtils.getServerInfo();
        String strParameters = getQueryString(req);
        String strErrorMessage = "";

        if (null == customMessage) {
            strErrorMessage = "".equals(getPrintStackTrace(e)) ? "NULL" : getPrintStackTrace(e);
        } else {
            strErrorMessage = customMessage;
        }

        saveErrorLogMap.put("srvrNm", strServerInfo[1]);
        saveErrorLogMap.put("srvrIpAddr", strServerInfo[0]);
        saveErrorLogMap.put("cntnUrlAddr", strErrorUri);
        saveErrorLogMap.put("errCn", StringUtils.getSubString(strErrorMessage, 4000));
        saveErrorLogMap.put("vrblCn", req.getMethod() + "\n" + strParameters);
        saveErrorLogMap.put("rgtrUnqId", userInfo.getUserId() + "");
        saveErrorLogMap.put("rgtrIpAddr", StringUtils.getRemoteIp());

        commonService.saveErrorLog(saveErrorLogMap);
    }

    private String getPrintStackTrace(Exception e) {
        StringWriter sw = new StringWriter();
        e.printStackTrace(new PrintWriter(sw));
        return sw.toString();
    }

    private String getQueryString(HttpServletRequest req) {
        Enumeration<String> eNames = req.getParameterNames();
        StringBuffer queryString = new StringBuffer();

        if (eNames.hasMoreElements()) {
            while (eNames.hasMoreElements()) {
                String name = (String) eNames.nextElement();
                String[] values = req.getParameterValues(name);

                if (values.length > 0) {
                    String value = values[0];
                    for (int i = 1; i < values.length; i++) {
                        value += "," + values[i];
                    }
                    queryString.append(name).append("=").append(value).append("&");
                }
            }
        }
        return queryString.toString();
    }
}
```

#### 3.3.4 CmmJsonResponse 개선

**파일: `com/fw/cmm/utils/CmmJsonResponse.java`**

```java
package com.fw.cmm.utils;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

import org.springframework.http.MediaType;
import org.springframework.security.web.DefaultRedirectStrategy;
import org.springframework.security.web.RedirectStrategy;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fw.cmm.dataSet.CustomMessage;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;

/**
 * 메뉴경로 : 공통
 * 프로그램개요 : FW에러 or GeneralException 관련 처리
 *                HTTP Status Code 지원 추가
 */
@Slf4j
public class CmmJsonResponse {

    private static RedirectStrategy redirectStrategy;
    private static String strMethod;

    /**
     * HTTP Status Code를 지정할 수 있는 메서드 (신규)
     */
    public static void returnJson(HttpServletRequest req, HttpServletResponse res,
                                   String strMsgCode, String strDetailMsg,
                                   String strUrl, int httpStatus) {

        Map<String, Object> map = new HashMap<>();
        String url = strUrl;

        if ("".equals(strUrl) || strUrl == null) {
            url = req.getContextPath() + "/";
        } else {
            url = req.getContextPath() + strUrl;
        }

        boolean isAjax = "XMLHttpRequest".equals(req.getHeader("X-Requested-With"));

        CustomMessage customMessage = new CustomMessage();
        customMessage.setMsgCd(strMsgCode);
        customMessage.setDetailMsg(strDetailMsg);
        customMessage.setUrl(url);

        try {
            redirectStrategy = new DefaultRedirectStrategy();
            strMethod = req.getMethod();

            // GET와 POST방식에 따라 리턴 형태를 다르게 함
            if ("GET".equals(strMethod)) {
                req.getSession().setAttribute("msg", customMessage.getDetailMsg());
                redirectStrategy.sendRedirect(req, res, url);
                return;
            } else {
                if (!isAjax) {
                    req.getSession().setAttribute("msg", customMessage.getDetailMsg());
                    redirectStrategy.sendRedirect(req, res, url);
                } else {
                    ObjectMapper mapper = new ObjectMapper();

                    // HTTP Status Code 적용
                    res.setStatus(httpStatus);
                    res.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    res.setCharacterEncoding("UTF-8");

                    map.put("message", customMessage);
                    res.getWriter().print(mapper.writeValueAsString(map));
                    res.getWriter().flush();
                    res.getWriter().close();
                }
                return;
            }
        } catch (IOException e) {
            log.error("================================================== Confirmation Required Start ==================================================");
            log.error("CmmJsonResponse IOException");
            log.error(e.getMessage());
            log.error("================================================== Confirmation Required END ==================================================");
        }
    }

    /**
     * 기존 메서드 (하위 호환성 유지 - 기본 200 OK)
     */
    @Deprecated
    public static void returnJson(HttpServletRequest req, HttpServletResponse res,
                                   String strMsgCode, String strDetailMsg, String strUrl) {
        returnJson(req, res, strMsgCode, strDetailMsg, strUrl, HttpServletResponse.SC_OK);
    }
}
```

---

### 3.4 Controller 적용 예시

#### 3.4.1 수정 전 (현재)

**파일: `FacilityInspectionResultRegistController.java`**

```java
@RestController
@RequestMapping("/inw/inr/fir")
public class FacilityInspectionResultRegistController extends BaseService {

    @Autowired
    private FacilityInspectionResultRegistService facilityInspectionResultRegistService;

    @PostMapping("/FacilityInspectionResultRegist/selectNmorList.do")
    public List<Map<String, Object>> selectNmorList(@RequestBody ParamMap param) {
        return facilityInspectionResultRegistService.selectNmorList(param);
    }

    @PostMapping("/FacilityInspectionResultRegist/insertNmorData.do")
    public DtoAssembler insertNmorData(@RequestBody ParamMap param) {
        int cnt = facilityInspectionResultRegistService.insertNmorData(param);
        return super.saveData("insert", Integer.valueOf(cnt));
    }
}
```

#### 3.4.2 수정 후 (개선)

**파일: `FacilityInspectionResultRegistController.java`**

```java
package com.biz.inw.inr.fir;

import com.biz.inw.inr.irr.InspectionResultRegistService;
import com.fw.cmm.dataSet.BaseController;  // BaseService → BaseController
import com.fw.cmm.dataSet.DtoAssembler;
import com.fw.cmm.exception.BadRequestException;
import com.fw.cmm.exception.NotFoundException;
import com.fw.cmm.utils.ParamMap;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;
import java.util.Map;

/**
 * 메뉴경로 : 검사결과 - 검사결과등록 - 시설검사결과등록
 * 프로그램개요 : 시설검사결과등록 관련정보 처리
 * 변경이력
 * ------------------------------------------------------------
 * 버전      개발일자       이름      설명(수정내용)
 * ------------------------------------------------------------
 * 1.0       2025. 7. 23.   노채은    최초 개발
 * 2.0       2025. 11. 11.  refactor  HTTP Status Code 적용
 */
@RestController
@RequestMapping("/inw/inr/fir")
public class FacilityInspectionResultRegistController extends BaseController {

    @Autowired
    private FacilityInspectionResultRegistService facilityInspectionResultRegistService;

    /**
     * 차수 목록 조회
     * @return 200 OK - 조회 성공
     */
    @PostMapping("/FacilityInspectionResultRegist/selectNmorList.do")
    public ResponseEntity<DtoAssembler> selectNmorList(@RequestBody ParamMap param) {
        List<Map<String, Object>> result = facilityInspectionResultRegistService.selectNmorList(param);
        return super.afterSelectList(result);  // 200 OK 반환
    }

    /**
     * 차수 추가
     * @return 201 Created - 생성 성공
     *         400 Bad Request - 유효성 검증 실패
     * @author ceno
     * @date 2025. 10. 27.
     */
    @PostMapping("/FacilityInspectionResultRegist/insertNmorData.do")
    public ResponseEntity<DtoAssembler> insertNmorData(@RequestBody ParamMap param) {

        // 유효성 검증 예시
        if (param == null || !param.containsKey("필수값")) {
            throw new BadRequestException("fw.cmm.036", new String[]{}); // 400 Bad Request
        }

        int cnt = facilityInspectionResultRegistService.insertNmorData(param);

        if (cnt == 0) {
            throw new BadRequestException("fw.cmm.042", new String[]{"등록"}); // 400 Bad Request
        }

        return super.afterInsert(Integer.valueOf(cnt));  // 201 Created 반환
    }

    /**
     * 차수 수정
     * @return 200 OK - 수정 성공
     *         404 Not Found - 대상 데이터 없음
     */
    @PostMapping("/FacilityInspectionResultRegist/updateNmorData.do")
    public ResponseEntity<DtoAssembler> updateNmorData(@RequestBody ParamMap param) {

        int cnt = facilityInspectionResultRegistService.updateNmorData(param);

        if (cnt == 0) {
            throw new NotFoundException("fw.cmm.043", new String[]{"수정 대상"}); // 404 Not Found
        }

        return super.afterUpdate(Integer.valueOf(cnt));  // 200 OK 반환
    }

    /**
     * 차수 삭제
     * @return 200 OK - 삭제 성공
     *         404 Not Found - 대상 데이터 없음
     */
    @PostMapping("/FacilityInspectionResultRegist/deleteNmorData.do")
    public ResponseEntity<DtoAssembler> deleteNmorData(@RequestBody ParamMap param) {

        int cnt = facilityInspectionResultRegistService.deleteNmorData(param);

        if (cnt == 0) {
            throw new NotFoundException("fw.cmm.043", new String[]{"삭제 대상"}); // 404 Not Found
        }

        return super.afterDelete(Integer.valueOf(cnt));  // 200 OK 반환
    }

    /**
     * 사업자 상태 조회 (외부 API)
     * @return 200 OK - 조회 성공
     *         400 Bad Request - 필수 파라미터 누락
     *         500 Internal Server Error - 외부 API 에러
     */
    @PostMapping("/FacilityInspectionResultRegist/getBrnoInfo.do")
    public ResponseEntity<DtoAssembler> getCommonDataJSON(@RequestBody ParamMap record) {

        if (record == null || !record.containsKey("RRNO")) {
            throw new BadRequestException("fw.cmm.036", new String[]{}); // 400 Bad Request
        }

        try {
            Map<String, Object> result = facilityInspectionResultRegistService.getBrnoInfo(record);

            if (result == null) {
                throw new NotFoundException("fw.cmm.044", new String[]{"사업자 정보"}); // 404 Not Found
            }

            return super.afterCommonData(result);  // 200 OK 반환

        } catch (Exception e) {
            log.error("외부 API 호출 에러", e);
            throw new RuntimeException("외부 API 호출 중 오류가 발생했습니다.");  // 500 Internal Server Error
        }
    }
}
```

#### 3.4.3 다양한 상황 예시

```java
@RestController
@RequestMapping("/api/example")
public class ExampleController extends BaseController {

    // 1. 조회 - 데이터 있음 (200 OK)
    @GetMapping("/users")
    public ResponseEntity<DtoAssembler> getUsers() {
        List<Map<String, Object>> users = userService.findAll();
        return super.afterSelectList(users);  // 200 OK
    }

    // 2. 조회 - 데이터 없음 (200 OK, 빈 배열)
    @GetMapping("/empty")
    public ResponseEntity<DtoAssembler> getEmptyList() {
        List<Map<String, Object>> empty = new ArrayList<>();
        return super.afterSelectList(empty);  // 200 OK, msgCd: "010"
    }

    // 3. 생성 - 성공 (201 Created)
    @PostMapping("/users")
    public ResponseEntity<DtoAssembler> createUser(@RequestBody ParamMap param) {
        int cnt = userService.create(param);
        return super.afterInsert(cnt);  // 201 Created
    }

    // 4. 생성 - 중복 에러 (409 Conflict)
    @PostMapping("/users/duplicate")
    public ResponseEntity<DtoAssembler> createDuplicateUser(@RequestBody ParamMap param) {
        boolean exists = userService.exists(param.getString("userId"));
        if (exists) {
            throw new ConflictException("fw.cmm.045", new String[]{"사용자 ID"});  // 409 Conflict
        }
        int cnt = userService.create(param);
        return super.afterInsert(cnt);
    }

    // 5. 수정 - 대상 없음 (404 Not Found)
    @PutMapping("/users/{id}")
    public ResponseEntity<DtoAssembler> updateUser(@PathVariable String id, @RequestBody ParamMap param) {
        int cnt = userService.update(id, param);
        if (cnt == 0) {
            throw new NotFoundException("fw.cmm.043", new String[]{"사용자"});  // 404 Not Found
        }
        return super.afterUpdate(cnt);  // 200 OK
    }

    // 6. 삭제 - 제약조건 위배 (409 Conflict)
    @DeleteMapping("/users/{id}")
    public ResponseEntity<DtoAssembler> deleteUser(@PathVariable String id) {
        boolean hasRelatedData = userService.hasRelatedData(id);
        if (hasRelatedData) {
            throw new ConflictException("fw.cmm.046", new String[]{"관련 데이터가 존재하여 삭제할 수 없습니다"});
        }
        int cnt = userService.delete(id);
        return super.afterDelete(cnt);  // 200 OK
    }

    // 7. 유효성 검증 실패 (400 Bad Request)
    @PostMapping("/validate")
    public ResponseEntity<DtoAssembler> validateData(@RequestBody ParamMap param) {
        if (!param.containsKey("requiredField")) {
            throw new BadRequestException("fw.cmm.036", new String[]{});  // 400 Bad Request
        }
        // 처리 로직...
        return ResponseEntity.ok(new DtoAssembler());
    }
}
```

---

## 4. 메시지 코드 추가

### 4.1 신규 메시지 코드

**파일: `messages.properties` 또는 DB 메시지 테이블**

```properties
# 기존 메시지
fw.cmm.001=등록되었습니다.
fw.cmm.003=수정되었습니다.
fw.cmm.005=삭제되었습니다.
fw.cmm.017={0}이(가) 없습니다.
fw.cmm.023={0}이(가) 완료되었습니다.
fw.cmm.031=처리되었습니다.
fw.cmm.033={0}오류가 발생했습니다. 관리자에게 문의하세요.
fw.cmm.036=필수값이 없습니다. 관리자에게 문의하세요

# 신규 추가 메시지
fw.cmm.040={0}을(를) 찾을 수 없습니다.
fw.cmm.041={0}이(가) 이미 존재합니다.
fw.cmm.042={0}에 실패했습니다.
fw.cmm.043={0}을(를) 찾을 수 없습니다.
fw.cmm.044={0} 조회에 실패했습니다.
fw.cmm.045={0}이(가) 중복되었습니다.
fw.cmm.046={0} 삭제할 수 없습니다.
```

---

## 5. 마이그레이션 전략

### 5.1 단계별 적용 계획

#### Phase 1: 인프라 구축 (1주)
- [ ] `BaseController` 클래스 생성 (`BaseService` 복사 후 수정)
- [ ] 새로운 Exception 클래스들 생성
- [ ] `GeneralExceptionHandler` 수정
- [ ] `CmmJsonResponse` 오버로딩 메서드 추가
- [ ] 신규 메시지 코드 등록

#### Phase 2: 파일럿 적용 (1주)
- [ ] 1~2개 Controller 선택하여 적용
- [ ] 프론트엔드 영향도 확인
- [ ] 로깅/모니터링 확인
- [ ] 문제점 파악 및 개선

#### Phase 3: 점진적 확대 (4주)
- [ ] 모듈별로 순차 적용 (주당 10~15개 Controller)
- [ ] 각 단계마다 테스트 및 검증
- [ ] 문제 발생 시 롤백 가능하도록 준비

#### Phase 4: 레거시 정리 (1주)
- [ ] `BaseService` → `BaseController`로 완전 전환
- [ ] `@Deprecated` 메서드 제거
- [ ] 문서화 완료

### 5.2 하위 호환성 보장

```java
// BaseService를 BaseController의 별칭으로 유지 (Deprecated)
@Deprecated
public class BaseService extends BaseController {
    // 기존 코드 그대로 동작
}
```

이렇게 하면 기존 코드 수정 없이도 점진적 마이그레이션 가능합니다.

### 5.3 롤백 계획

각 Phase별로 Git 브랜치 분리:
```
main
├── feature/http-status-phase1 (인프라)
├── feature/http-status-phase2 (파일럿)
├── feature/http-status-phase3-module-A
├── feature/http-status-phase3-module-B
└── ...
```

문제 발생 시 해당 브랜치만 롤백 가능.

---

## 6. 테스트 전략

### 6.1 단위 테스트

```java
@SpringBootTest
@AutoConfigureMockMvc
class FacilityInspectionResultRegistControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @DisplayName("차수 추가 성공 - 201 Created 반환")
    void insertNmorData_Success_Returns201() throws Exception {
        ParamMap param = new ParamMap();
        param.put("필수값", "테스트");

        mockMvc.perform(post("/inw/inr/fir/FacilityInspectionResultRegist/insertNmorData.do")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(param)))
                .andExpect(status().isCreated())  // 201 검증
                .andExpect(jsonPath("$.message.msgCd").value("100"))
                .andExpect(jsonPath("$.message.detailMsg").exists());
    }

    @Test
    @DisplayName("차수 추가 실패 - 필수값 없음 - 400 Bad Request")
    void insertNmorData_MissingRequired_Returns400() throws Exception {
        ParamMap param = new ParamMap();
        // 필수값 없음

        mockMvc.perform(post("/inw/inr/fir/FacilityInspectionResultRegist/insertNmorData.do")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(param)))
                .andExpect(status().isBadRequest())  // 400 검증
                .andExpect(jsonPath("$.message.msgCd").value("110"));
    }

    @Test
    @DisplayName("차수 수정 실패 - 대상 없음 - 404 Not Found")
    void updateNmorData_NotFound_Returns404() throws Exception {
        ParamMap param = new ParamMap();
        param.put("id", "존재하지않는ID");

        mockMvc.perform(post("/inw/inr/fir/FacilityInspectionResultRegist/updateNmorData.do")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(param)))
                .andExpect(status().isNotFound())  // 404 검증
                .andExpect(jsonPath("$.message.msgCd").value("110"));
    }

    @Test
    @DisplayName("차수 조회 성공 - 200 OK 반환")
    void selectNmorList_Success_Returns200() throws Exception {
        ParamMap param = new ParamMap();

        mockMvc.perform(post("/inw/inr/fir/FacilityInspectionResultRegist/selectNmorList.do")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(param)))
                .andExpect(status().isOk())  // 200 검증
                .andExpect(jsonPath("$.message.msgCd").value("000"))
                .andExpect(jsonPath("$.list").isArray());
    }
}
```

### 6.2 통합 테스트

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class HttpStatusIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void 전체_CRUD_시나리오_HTTP_Status_검증() {
        // 1. 생성 - 201 Created
        ResponseEntity<Map> createResponse = restTemplate.postForEntity(
            "/api/users",
            createRequest,
            Map.class
        );
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        // 2. 조회 - 200 OK
        ResponseEntity<Map> getResponse = restTemplate.getForEntity(
            "/api/users/1",
            Map.class
        );
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);

        // 3. 수정 - 200 OK
        ResponseEntity<Map> updateResponse = restTemplate.exchange(
            "/api/users/1",
            HttpMethod.PUT,
            updateRequest,
            Map.class
        );
        assertThat(updateResponse.getStatusCode()).isEqualTo(HttpStatus.OK);

        // 4. 삭제 - 200 OK
        ResponseEntity<Map> deleteResponse = restTemplate.exchange(
            "/api/users/1",
            HttpMethod.DELETE,
            null,
            Map.class
        );
        assertThat(deleteResponse.getStatusCode()).isEqualTo(HttpStatus.OK);

        // 5. 삭제된 데이터 조회 - 404 Not Found
        ResponseEntity<Map> notFoundResponse = restTemplate.getForEntity(
            "/api/users/1",
            Map.class
        );
        assertThat(notFoundResponse.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
    }
}
```

---

## 7. 프론트엔드 영향도 분석

### 7.1 현재 프론트엔드 처리 방식

```javascript
// 기존 방식 (항상 200 OK)
axios.post('/api/data', params)
    .then(response => {
        // response.status는 항상 200
        if (response.data.message.msgCd === '000') {
            // 성공 처리
        } else if (response.data.message.msgCd === '110') {
            // 에러 처리
        }
    })
    .catch(error => {
        // 네트워크 에러만 여기서 처리
    });
```

### 7.2 개선된 방식 (하위 호환성 유지)

```javascript
// 개선 방식 (HTTP Status 활용 + 기존 msgCd도 유지)
axios.post('/api/data', params)
    .then(response => {
        // response.status가 200, 201 등으로 구분됨
        // 하지만 기존 msgCd도 여전히 존재
        if (response.data.message.msgCd === '000' || response.data.message.msgCd === '100') {
            // 성공 처리
            if (response.status === 201) {
                console.log('새로 생성됨');
            }
        }
    })
    .catch(error => {
        // HTTP Status 400, 404, 500 등이 여기로 옴
        if (error.response) {
            switch(error.response.status) {
                case 400:
                    alert('잘못된 요청입니다.');
                    break;
                case 404:
                    alert('데이터를 찾을 수 없습니다.');
                    break;
                case 500:
                    alert('서버 오류가 발생했습니다.');
                    break;
            }
            // 기존 msgCd도 여전히 사용 가능
            console.log(error.response.data.message.detailMsg);
        }
    });
```

### 7.3 점진적 개선

기존 코드는 수정 없이 동작하며, 새로운 기능 개발 시에만 HTTP Status 활용:

```javascript
// 공통 Axios 인터셉터 추가
axios.interceptors.response.use(
    response => {
        // 성공 응답 (200, 201 등)
        return response;
    },
    error => {
        // 에러 응답 (400, 404, 500 등)
        if (error.response) {
            // 새로운 방식: HTTP Status 기반 처리
            const status = error.response.status;
            const message = error.response.data?.message?.detailMsg || '오류가 발생했습니다.';

            // 전역 에러 처리
            if (status >= 500) {
                showErrorNotification('서버 오류: ' + message);
            } else if (status === 404) {
                showErrorNotification('데이터를 찾을 수 없습니다: ' + message);
            } else if (status === 400) {
                showErrorNotification('잘못된 요청: ' + message);
            }
        }
        return Promise.reject(error);
    }
);
```

---

## 8. 모니터링 및 로깅 개선

### 8.1 Access Log에서 HTTP Status 활용

```
# 기존 (모든 요청이 200)
2025-11-11 10:00:00 POST /api/users 200 150ms
2025-11-11 10:00:01 POST /api/users 200 120ms  <- 실제는 에러인데 200
2025-11-11 10:00:02 GET  /api/users/999 200 80ms  <- 실제는 404인데 200

# 개선 후 (정확한 상태 코드)
2025-11-11 10:00:00 POST /api/users 201 150ms  <- 생성 성공
2025-11-11 10:00:01 POST /api/users 409 120ms  <- 중복 에러
2025-11-11 10:00:02 GET  /api/users/999 404 80ms  <- Not Found
```

### 8.2 모니터링 대시보드

```
# Prometheus/Grafana 메트릭
http_requests_total{status="2xx"} 850
http_requests_total{status="4xx"} 120  <- 클라이언트 에러
http_requests_total{status="5xx"} 30   <- 서버 에러

# 에러율 계산 가능
error_rate = (4xx + 5xx) / total = 150 / 1000 = 15%
```

### 8.3 로그 필터링 개선

```java
@Slf4j
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {

        long startTime = System.currentTimeMillis();

        filterChain.doFilter(request, response);

        long duration = System.currentTimeMillis() - startTime;
        int status = response.getStatus();

        // HTTP Status에 따라 로그 레벨 구분
        if (status >= 500) {
            log.error("[{}] {} {} - {}ms - SERVER ERROR",
                status, request.getMethod(), request.getRequestURI(), duration);
        } else if (status >= 400) {
            log.warn("[{}] {} {} - {}ms - CLIENT ERROR",
                status, request.getMethod(), request.getRequestURI(), duration);
        } else {
            log.info("[{}] {} {} - {}ms",
                status, request.getMethod(), request.getRequestURI(), duration);
        }
    }
}
```

---

## 9. 기대 효과

### 9.1 개발 측면
- ✅ RESTful API 원칙 준수
- ✅ HTTP 표준 활용으로 학습 곡선 감소
- ✅ API 문서 가독성 향상
- ✅ 에러 처리 일관성 확보

### 9.2 운영 측면
- ✅ 모니터링 정확도 향상
- ✅ 에러 추적 용이
- ✅ 로그 필터링 효율화
- ✅ SLA 측정 가능

### 9.3 클라이언트 측면
- ✅ HTTP 레벨에서 에러 핸들링 가능
- ✅ 표준 HTTP 클라이언트 기능 활용
- ✅ 캐싱 전략 수립 가능
- ✅ 에러 메시지 명확화

---

## 10. 리스크 및 대응 방안

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| 프론트엔드 호환성 문제 | 중 | 기존 msgCd 유지, 점진적 마이그레이션 |
| 개발자 학습 비용 | 하 | 가이드 문서 작성, 예제 코드 제공 |
| 마이그레이션 기간 혼재 | 중 | BaseService 별칭 유지, 명확한 단계 구분 |
| 예상치 못한 버그 | 중 | 충분한 테스트, 모듈별 점진 적용 |
| 성능 영향 | 하 | ResponseEntity는 오버헤드 거의 없음 |

---

## 11. 체크리스트

### Phase 1 완료 조건
- [ ] BaseController 클래스 생성 및 테스트
- [ ] Exception 클래스들 생성
- [ ] GeneralExceptionHandler 수정
- [ ] 신규 메시지 코드 등록
- [ ] 단위 테스트 작성

### Phase 2 완료 조건
- [ ] 파일럿 Controller 2개 적용
- [ ] 프론트엔드 동작 확인
- [ ] 통합 테스트 통과
- [ ] 로그 확인

### Phase 3 완료 조건
- [ ] 전체 Controller의 80% 이상 적용
- [ ] 회귀 테스트 통과
- [ ] 성능 테스트 통과

### Phase 4 완료 조건
- [ ] BaseService Deprecated 제거
- [ ] 레거시 코드 정리
- [ ] 문서화 완료
- [ ] 팀 교육 완료

---

## 12. 참고 자료

- [Spring ResponseEntity 공식 문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/ResponseEntity.html)
- [HTTP Status Code RFC 9110](https://httpwg.org/specs/rfc9110.html#status.codes)
- [RESTful API 설계 가이드](https://restfulapi.net/http-status-codes/)
- [Spring @RestControllerAdvice](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/RestControllerAdvice.html)

---

## 13. 문의 및 지원

- 기술 문의: 개발팀 리드
- 마이그레이션 지원: 아키텍처팀
- 긴급 이슈: 프로젝트 매니저

---

**작성일**: 2025-11-11
**작성자**: Refactoring Team
**버전**: 1.0
