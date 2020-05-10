# 구현

### API

* 기능 : 고양이의 연구 참여를 서버에 저장한다.
* code : [api](https://github.com/JustBurrow/street-cat-study-code/tree/master/api)

보안을 생각하면 많은 추가기능이 필요하겠지만, 다 생략하고 최소한의 정보만 남긴다.

#### Request

인식표ID\(chipId\)와 장비ID\(deviceId\)를 전송한다.

code : [AddRequest](https://github.com/JustBurrow/street-cat-study-code/blob/master/api/src/main/java/kr/lul/street/cat/study/api/rest/controller/request/AddRequest.java)

```java
public class AddRequest {
  private UUID chipId;
  private UUID deviceId;
  // 생략
}
```

#### Response

고양이가 언제 참가했는지 정보를 반환한다.

code : [AddResponse](https://github.com/JustBurrow/street-cat-study-code/blob/master/api/src/main/java/kr/lul/street/cat/study/api/rest/controller/response/AddResponse.java)

```java
public class AddResponse {
  private UUID deviceId;
  private UUID chipId;
  private LocalDateTime createdAt;
  // 생략
}
```

#### Controller

연구 참가 정보를 받아서 등록하거나 등록된 정보 중에서 필요한 정보만 모아서 응답한다.

code : [JoinRestController](https://github.com/JustBurrow/street-cat-study-code/blob/master/api/src/main/java/kr/lul/street/cat/study/api/rest/controller/JoinRestController.java)

```java
@RestController
@RequestMapping
public class JoinRestController {
  @Autowired
  private CatService service;

  @PostMapping("/join/add")
  @ResponseBody
  public AddResponse add(@RequestBody AddRequest request) {
    Cat cat = this.service.add(request.getChipId(), request.getDeviceId());
    AddResponse response = new AddResponse(cat.getDeviceId(), cat.getChipId(), cat.getCreatedAt());
    return response;
  }
}
```

#### Service

고양이의 연구 참여 정보가 이미 있는지 확인해서 있으면 기존 정보를, 없으면 신규 등록하고 그 정보를 반환한다. 신규 등록할 때는 고양이 정보 관리 시스템에서 고양이 이름을 확인한다.

code : [CatService](https://github.com/JustBurrow/street-cat-study-code/blob/master/api/src/main/java/kr/lul/street/cat/study/api/service/CatService.java)

```java
@Service
public class CatService {
  @Autowired
  private CatManagerService catManagerService;
  @Autowired
  private CatRepository repository;
  
  public Cat add(UUID chipId, UUID deviceId) {
    Cat cat = this.repository.findByChipIdAndDeviceId(chipId, deviceId);
    if (null == cat) {
      CatInfo catInfo = this.catManagerService.catInfo(chipId);

      cat = new Cat(chipId, deviceId, catInfo.getName());
      cat = this.repository.save(cat);
    }
    return cat;
  }
}
```

### Batch

* 기능 : 장비가 업로드한 csv 파일을 읽어서 고양이가 장비를 사용한 내역을 고양이 관리 시스템에 저장한다.

