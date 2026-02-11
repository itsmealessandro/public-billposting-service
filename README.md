# Public Billposting Booking System

## Project overview
This repository contains a public billposting booking system built on Camunda BPM embedded in a Spring Boot application. The process is modeled in BPMN and orchestrates REST and SOAP services to select zones, handle confirmations or cancellations, and persist confirmed orders.

## Repository structure
- `camundaEngine/camundaApi`: Spring Boot + Camunda application (main service).
- `services/`: Prebuilt JARs for user, zones, and posting services plus `run_services.sh`.
- `postingService/`, `userService/`, `zonesService/`: Dockerfiles and JARs for containers.
- `compose.yaml`: Docker Compose for the full stack.
- `persistent_data/`: Mounted data directory for confirmed orders.
- `verify_cancel.sh`, `verify_error.sh`: Smoke checks via curl.

## Technologies
- Java 8
- Spring Boot 2.7.x
- Camunda BPM 7.19
- REST and SOAP
- Maven

## External services
You can run the prebuilt services locally from the `services/` directory:

```bash
java -jar services/user-service.jar
java -jar services/zones-service.jar
java -jar services/posting-service.jar
```

### External service endpoints
| Service | Type | Endpoint / WSDL | Description |
| --- | --- | --- | --- |
| `user-service` | REST | `GET http://localhost:9080/user/{username}` | Gets user information |
| `zones-service` | REST | `GET http://localhost:9090/zones/{format}` | Gets zone availability |
| `posting-service` | SOAP | `http://localhost:8888/postingservice/postingservice.wsdl` | Handles posting requests |

## Run locally
1. Start the external services:
   ```bash
   ./services/run_services.sh
   ```
2. Run the Camunda app:
   ```bash
   mvn -f camundaEngine/camundaApi/pom.xml spring-boot:run
   ```
3. Open Camunda:
   - Dashboard: http://localhost:8080

## Run with Docker
```bash
docker compose -f compose.yaml up --build
```

Networking note: from inside containers use service names and ports, e.g. `http://posting-service:8888/postingservice`.

## API endpoints

### POST /api/booking/request
Starts the booking process.

Request:
```json
{
  "username": "mariorossi",
  "cities": ["L'Aquila", "Rome"],
  "format": "60x80",
  "algorithm": "greedy",
  "maxPrices": {"L'Aquila": 30.0, "Rome": 40.0}
}
```

Response:
```json
{
  "requestId": "03NzgXNyQE",
  "selectedZones": {
    "L'Aquila": ["Zone1", "Zone2"],
    "Rome": ["Zone3"]
  },
  "totalPrice": 29.5,
  "businessKey": "3bdce3da-8f8e-4cc2-9fe7-2c3a9a4f1cc2"
}
```

### POST /api/booking/decision
Confirms or cancels a booking.

Request:
```json
{
  "requestId": "03NzgXNyQE",
  "decision": "CANCEL",
  "businessKey": "3bdce3da-8f8e-4cc2-9fe7-2c3a9a4f1cc2"
}
```

Response:
```json
{
  "message": "Decision processed for request: 03NzgXNyQE",
  "invoiceNumber": "2057718223",
  "amountDue": 29.5,
  "status": "CANCEL"
}
```

### GET /api/booking/history
Returns a list of historic process instances and variables.

## Process workflow
1. Receive a booking request.
2. Fetch user data from `user-service` and zones from `zones-service`.
3. Select zones by algorithm and budget.
4. Submit posting request to `posting-service` (SOAP).
5. Return selected zones, request ID, and business key.
6. Await user decision and finalize or cancel the posting.

## File persistence
Confirmed orders are stored in `persistent_data/` (mounted by Docker). A consolidated `posting_requests.txt` may be written depending on process logic.

## Error handling
- Malformed JSON returns `400` with `ErrorResponse`.
- Validation errors throw `IllegalArgumentException` and return `400`.
- Unexpected failures return `500` with `ErrorResponse`.

## Build and test
- Build:
  `mvn -f camundaEngine/camundaApi/pom.xml clean package`
- Test (when tests exist):
  `mvn -f camundaEngine/camundaApi/pom.xml test`
- Single test class:
  `mvn -f camundaEngine/camundaApi/pom.xml -Dtest=MyTest test`
- Single test method:
  `mvn -f camundaEngine/camundaApi/pom.xml -Dtest=MyTest#myTest test`

## Notes
This project was developed for the Business Process Development course at the University of L'Aquila.
