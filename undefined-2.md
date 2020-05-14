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

#### 업로드 디렉토리 모니터

장비가 데이터 파일을 업로드 하는 디렉토리를 모니터링 하다가, 새로운 파일이 생기면 처리 로직을 호출한다.

`postConstruct()`에서 바로 모니터링 루프를 돌면 메서드 실행이 끝나지 않기 때문에, 새로 쓰레드를 만들어서 실행한다.

code : [InputDirectoryMonitor](https://github.com/JustBurrow/street-cat-study-code/blob/master/batch/src/main/java/kr/lul/street/cat/study/batch/monitor/InputDirectoryMonitor.java)

```java
@Component
public class InputDirectoryMonitor {
  @Autowired
  private BatchProperties properties;

  @Autowired
  private BatchService service;

  private WatchService watchService;
  private Thread listener;

  @PostConstruct
  private void postConstruct() throws IOException {
    this.watchService = FileSystems.getDefault().newWatchService();
    Path path = Paths.get(this.properties.getInputDirectory().toURI());

    path.register(this.watchService, ENTRY_CREATE, ENTRY_MODIFY);

    this.listener = new Thread(this::monitor, InputDirectoryMonitor.class.getName() + ".listener");
    this.listener.start();
  }

  @PreDestroy
  private void preDestroy() {
    this.listener.interrupt();
  }

  private void monitor() {
    WatchKey key;
    try {
      while ((key = this.watchService.take()) != null) {
        key.pollEvents()
            .forEach(event -> {
              File file = new File(this.properties.getInputDirectory(), event.context().toString());
              Future<BatchResult> result = this.service.process(file);
            });
        key.reset();
      }
    } catch (Exception e) {
      throw new RuntimeException(e);
    }
  }
}
```

#### CSV 로딩

장비가 업로드한 csv 파일을 읽어 인스턴스로 변환한다.

code : [UseDataReader](https://github.com/JustBurrow/street-cat-study-code/blob/master/batch/src/main/java/kr/lul/street/cat/study/batch/service/UseDataReader.java)

```java
@Service
public class UseDataReader {
  private CSVFormat format;
  private DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

  @PostConstruct
  private void postConstruct() {
    this.format = CSVFormat.DEFAULT;
  }

  public List<UseData> load(File file) {
    try (BufferedReader reader = Files.newBufferedReader(file.toPath())) {
      return stream(this.format.parse(reader).spliterator(), false)
                 .map(record -> new UseData(
                     UUID.fromString(record.get(0)),
                     UUID.fromString(record.get(1)),
                     DeviceType.valueOf(parseInt(record.get(2))),
                     parseInt(record.get(3)),
                     LocalDateTime.parse(record.get(4), this.dateTimeFormatter)))
                 .collect(toList());
    } catch (IOException e) {
      throw new RuntimeException(e);
    }
  }
}
```

### 사용량 JPA 엔티티

장비 사용량 정보를 저장할 때 사용할 엔티티.

code : [Use](https://github.com/JustBurrow/street-cat-study-code/blob/master/batch/src/main/java/kr/lul/street/cat/study/batch/data/Use.java)

```java
@Entity(name = "Use")
@Table(name = "uses",
    indexes = @Index(name = "idx_uses", columnList = "chip_id ASC, device_id ASC, measured_at ASC"))
public class Use {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  @Column(name = "id", nullable = false, insertable = false, updatable = false)
  private long id;
  @Column(name = "chip_id", nullable = false, updatable = false)
  private UUID chipId;
  @Column(name = "device_id", nullable = false, updatable = false)
  private UUID deviceId;
  @Column(name = "type", nullable = false, updatable = false)
  @Enumerated(EnumType.STRING)
  private DeviceType type;
  @Column(name = "value", nullable = false, updatable = false)
  private int value;
  @Column(name = "measured_at", nullable = false, updatable = false)
  private LocalDateTime measuredAt;
  @Column(name = "created_at", nullable = false, updatable = false)
  private LocalDateTime createdAt;

  public Use() {// JPA only
  }

  public Use(UUID chipId, UUID deviceId, DeviceType type, int value, LocalDateTime measuredAt) {
    this.chipId = chipId;
    this.deviceId = deviceId;
    this.type = type;
    this.value = value;
    this.measuredAt = measuredAt;
  }

  @PrePersist
  private void prePersist() {
    this.createdAt = LocalDateTime.now().withNano(0);
  }

// 생략 ...

  @Override
  public String toString() {
    return new StringBuilder(Use.class.getSimpleName())
               .append("{id=").append(this.id)
               .append(", chipId=").append(this.chipId)
               .append(", deviceId=").append(this.deviceId)
               .append(", type=").append(this.type)
               .append(", value=").append(this.value)
               .append(", measuredAt=").append(this.measuredAt)
               .append(", createdAt=").append(this.createdAt)
               .append('}').toString();
  }
}
```

### 배치 로직

파일에서 읽은 데이터를 검증한 후 DB에 저장한다.

code : [BatchService](https://github.com/JustBurrow/street-cat-study-code/blob/master/batch/src/main/java/kr/lul/street/cat/study/batch/service/BatchService.java)

```java
@Service
public class BatchService {
  @Autowired
  private UseDataReader reader;
  @Autowired
  private ChipService chipService;
  @Autowired
  private DeviceService deviceService;
  @Autowired
  private UseRepository useRepository;

  @Async
  public Future<BatchResult> process(File file) {
    Instant start = Instant.now();
    AtomicInteger dataCounter = new AtomicInteger();
    AtomicInteger validCounter = new AtomicInteger();
    this.reader.load(file)
        .stream()
        .peek(useData -> dataCounter.incrementAndGet())
        .filter(useData -> this.chipService.validate(useData.getChipId()))
        .filter(useData -> this.deviceService.validate(useData.getDeviceId()))
        .forEach(useData -> {
          Use use = new Use(useData.getChipId(), useData.getDeviceId(),
              useData.getType(), useData.getValue(),
              useData.getMeasuredAt());
          use = this.useRepository.save(use);
          int i = validCounter.incrementAndGet();
        });

    BatchResult result = new BatchResult(file, dataCounter.get(), validCounter.get(), Duration.between(start, Instant.now()));
    return new AsyncResult<>(result);
  }
}
```

