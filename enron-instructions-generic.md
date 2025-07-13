# Enron to DaProject Test Data - Complete Setup Guide

*Version 1.0 - Last Updated: 2025-07-13*
*Carson Sweet, assisted by Claude AI*

This guide provides step-by-step instructions for transforming the Enron email dataset into Gmail API format test data for DaProject integration testing.

## Prerequisites

1. **Enron Email Dataset**: Download from [here](https://www.cs.cmu.edu/~enron/) or Kaggle
2. **Go 1.21+**: Required to run the transformer
3. **DaProject Project**: Your local DaProject development environment

## Step 1: Set Up the Transformer

### 1.1 Save the Transformer Code

```bash
# Create a tools directory in your DaProject project
cd ~/projects/da-project
mkdir -p tools/enron-transformer
cd tools/enron-transformer

# Save the enron_transformer.go file here
# (Copy the code from the previous artifact)
```

### 1.2 Initialize Go Module

```bash
go mod init github.com/da-project/enron-transformer
go mod tidy
```

## Step 2: Run the Transformer

### 2.1 Basic Usage

```bash
# Build the tool
go build -o enron-transformer enron_transformer.go

# Run with default settings (kaminski-v user, 5000 emails)
./enron-transformer \
  --enron-path ~/Downloads/maildir \
  --output ./test-data
```

### 2.2 Advanced Options

```bash
# Process a different user with custom limit
./enron-transformer \
  --enron-path ~/Downloads/maildir \
  --user kean-s \
  --limit 3000 \
  --test-email john.doe@example.com \
  --output ./test-data-kean

# Other good test users:
# - kaminski-v (4,500 emails, quantitative analyst)
# - kean-s (6,000 emails, government relations)
# - germany-c (3,800 emails, good mix)
# - dasovich-j (28,000 emails, California crisis period)
```

### 2.3 Output Files

After running, you'll have these files in your output directory:

```
test-data/
├── gmail_messages.json         # Full Gmail API message format
├── list_messages_response.json # Gmail list API response format
├── test_metadata.json         # Dataset statistics
└── transform_stats.json       # Transformation details
```

## Step 3: Create Gmail API Emulator

### 3.1 Create the Emulator Service

```go
// test/emulator/gmail_emulator.go
package emulator

import (
    "encoding/json"
    "fmt"
    "io/ioutil"
    "net/http"
    "strconv"
    "strings"
    
    "github.com/gorilla/mux"
)

type GmailEmulator struct {
    messages      map[string]*GmailMessage
    messageList   []MessageRef
    userEmail     string
}

func NewGmailEmulator(dataPath, userEmail string) (*GmailEmulator, error) {
    emulator := &GmailEmulator{
        messages:  make(map[string]*GmailMessage),
        userEmail: userEmail,
    }
    
    // Load messages
    messagesData, err := ioutil.ReadFile(filepath.Join(dataPath, "gmail_messages.json"))
    if err != nil {
        return nil, err
    }
    
    var messages []*GmailMessage
    if err := json.Unmarshal(messagesData, &messages); err != nil {
        return nil, err
    }
    
    // Index messages
    for _, msg := range messages {
        emulator.messages[msg.Id] = msg
    }
    
    // Load list response
    listData, err := ioutil.ReadFile(filepath.Join(dataPath, "list_messages_response.json"))
    if err != nil {
        return nil, err
    }
    
    var listResp ListMessagesResponse
    if err := json.Unmarshal(listData, &listResp); err != nil {
        return nil, err
    }
    
    emulator.messageList = listResp.Messages
    
    return emulator, nil
}

func (e *GmailEmulator) Start(port int) error {
    r := mux.NewRouter()
    
    // Gmail API endpoints
    r.HandleFunc("/gmail/v1/users/{userId}/messages", e.listMessages).Methods("GET")
    r.HandleFunc("/gmail/v1/users/{userId}/messages/{messageId}", e.getMessage).Methods("GET")
    
    fmt.Printf("Gmail Emulator started on port %d\n", port)
    return http.ListenAndServe(fmt.Sprintf(":%d", port), r)
}

func (e *GmailEmulator) listMessages(w http.ResponseWriter, r *http.Request) {
    pageToken := r.URL.Query().Get("pageToken")
    maxResults := r.URL.Query().Get("maxResults")
    q := r.URL.Query().Get("q")
    
    limit := 100
    if maxResults != "" {
        if n, err := strconv.Atoi(maxResults); err == nil {
            limit = n
        }
    }
    
    start := 0
    if pageToken != "" {
        if n, err := strconv.Atoi(pageToken); err == nil {
            start = n
        }
    }
    
    // Filter by query if provided
    filtered := e.filterMessages(q)
    
    end := start + limit
    if end > len(filtered) {
        end = len(filtered)
    }
    
    response := ListMessagesResponse{
        Messages:           filtered[start:end],
        ResultSizeEstimate: len(filtered),
    }
    
    if end < len(filtered) {
        response.NextPageToken = strconv.Itoa(end)
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

func (e *GmailEmulator) getMessage(w http.ResponseWriter, r *http.Request) {
    messageId := mux.Vars(r)["messageId"]
    
    msg, ok := e.messages[messageId]
    if !ok {
        http.Error(w, "Message not found", 404)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(msg)
}
```

### 3.2 Create Docker Compose for Testing

```yaml
# test/docker-compose.test.yml
version: '3.8'

services:
  gmail-emulator:
    build:
      context: .
      dockerfile: test/emulator/Dockerfile
    ports:
      - "8080:8080"
    volumes:
      - ./test-data:/data
    environment:
      - TEST_USER_EMAIL=test@example.com
      - DATA_PATH=/data

  postgres-test:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: footprints_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5433:5432"

  redis-test:
    image: redis:7-alpine
    ports:
      - "6380:6379"
```

## Step 4: Wire Into Integration Tests

### 4.1 Create Test Configuration

```go
// test/integration/config.go
package integration

type TestConfig struct {
    GmailEmulatorURL string
    TestUserEmail    string
    TestDataPath     string
}

func LoadTestConfig() *TestConfig {
    return &TestConfig{
        GmailEmulatorURL: getEnv("GMAIL_EMULATOR_URL", "http://localhost:8080"),
        TestUserEmail:    getEnv("TEST_USER_EMAIL", "test@example.com"),
        TestDataPath:     getEnv("TEST_DATA_PATH", "./test-data"),
    }
}
```

### 4.2 Create Base Test Suite

```go
// test/integration/suite_test.go
package integration

import (
    "context"
    "testing"
    "time"
    
    "github.com/stretchr/testify/suite"
)

type IntegrationTestSuite struct {
    suite.Suite
    config   *TestConfig
    emulator *GmailEmulator
    services *ServiceContainer
}

func (s *IntegrationTestSuite) SetupSuite() {
    s.config = LoadTestConfig()
    
    // Start Gmail emulator
    s.emulator = NewGmailEmulator(s.config.TestDataPath, s.config.TestUserEmail)
    go s.emulator.Start(8080)
    
    // Wait for emulator to be ready
    s.Eventually(func() bool {
        resp, err := http.Get(s.config.GmailEmulatorURL + "/health")
        return err == nil && resp.StatusCode == 200
    }, 10*time.Second, 100*time.Millisecond)
    
    // Start DaProject services
    s.services = StartTestServices(ServiceConfig{
        GmailEndpoint: s.config.GmailEmulatorURL,
    })
}

func TestIntegrationSuite(t *testing.T) {
    suite.Run(t, new(IntegrationTestSuite))
}
```

### 4.3 Create Specific Test Cases

```go
// test/integration/ingestion_test.go
package integration

func (s *IntegrationTestSuite) TestFullIngestionPipeline() {
    ctx := context.Background()
    userID := "test-user-123"
    
    // Create user environment
    err := s.services.UserEnv.CreateEnvironment(ctx, userID)
    s.Require().NoError(err)
    
    // Start ingestion
    err = s.services.Orchestration.StartIngestion(ctx, &IngestionRequest{
        UserID:      userID,
        AccessToken: "fake-token", // Emulator doesn't validate
        StartDate:   time.Now().AddDate(-3, 0, 0),
        EndDate:     time.Now(),
    })
    s.Require().NoError(err)
    
    // Wait for completion
    s.Eventually(func() bool {
        status, _ := s.services.Orchestration.GetStatus(ctx, userID)
        return status.State == "completed"
    }, 5*time.Minute, 5*time.Second)
    
    // Verify results
    stats, err := s.services.Analytics.GetStats(ctx, userID)
    s.Require().NoError(err)
    s.Assert().Equal(5000, stats.MessagesProcessed)
    s.Assert().Greater(stats.EntitiesExtracted, 100)
    s.Assert().Greater(stats.RelationshipsFound, 20)
}

func (s *IntegrationTestSuite) TestTimelineGeneration() {
    ctx := context.Background()
    userID := "test-user-123"
    
    // Assume ingestion is complete
    
    timeline, err := s.services.Timeline.Generate(ctx, userID)
    s.Require().NoError(err)
    
    // Verify timeline structure
    s.Assert().Greater(len(timeline.Months), 30) // 3+ years of data
    
    // Check a specific month has content
    for _, month := range timeline.Months {
        if month.Year == 2022 && month.Month == 3 {
            s.Assert().NotEmpty(month.Summary)
            s.Assert().Greater(len(month.Events), 0)
            s.Assert().Greater(len(month.People), 0)
        }
    }
}

func (s *IntegrationTestSuite) TestRelationshipAnalysis() {
    ctx := context.Background()
    userID := "test-user-123"
    
    relationships, err := s.services.Relationships.Analyze(ctx, userID)
    s.Require().NoError(err)
    
    // Find Sarah Chen (most frequent contact in test data)
    var sarahRel *Relationship
    for _, rel := range relationships {
        if rel.Person.Email == "sarah.chen@gmail.com" {
            sarahRel = rel
            break
        }
    }
    
    s.Require().NotNil(sarahRel)
    s.Assert().Equal("sister", sarahRel.Type)
    s.Assert().Greater(sarahRel.Strength, 0.8)
    s.Assert().Greater(len(sarahRel.Evolution), 20) // Many data points
}
```

## Step 5: Run Tests

### 5.1 Start Test Environment

```bash
# Terminal 1: Start test databases
docker-compose -f test/docker-compose.test.yml up -d postgres-test redis-test

# Terminal 2: Start Gmail emulator
cd tools/enron-transformer
./enron-transformer --enron-path ~/Downloads/maildir --output ../../test/test-data

cd ../..
go run test/emulator/main.go --data ./test/test-data

# Terminal 3: Run tests
go test ./test/integration/... -v
```

### 5.2 Run Specific Test Scenarios

```bash
# Test only ingestion
go test ./test/integration -run TestFullIngestionPipeline -v

# Test with different dataset
./enron-transformer --user dasovich-j --limit 1000 --output ./test-data-crisis
go test ./test/integration -run TestCrisisPeriod -v

# Benchmark processing speed
go test ./test/integration -run BenchmarkIngestion -bench=.
```

## Step 6: Use in CI/CD

### 6.1 GitHub Actions Workflow

```yaml
# .github/workflows/integration-test.yml
name: Integration Tests

on:
  pull_request:
  push:
    branches: [main]

jobs:
  integration-test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'
    
    - name: Download test data
      run: |
        # Pre-generated test data stored in GCS
        gsutil cp gs://DaProject-test-data/enron-5k.tar.gz .
        tar -xzf enron-5k.tar.gz -C test/
    
    - name: Start services
      run: |
        docker-compose -f test/docker-compose.test.yml up -d
        ./scripts/wait-for-services.sh
    
    - name: Run integration tests
      run: |
        go test ./test/integration/... -v -timeout 30m
    
    - name: Upload test results
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: test-results
        path: test/results/
```

## Step 7: Visualize Test Results

### 7.1 Generate Test Reports

```go
// test/reports/visualizer.go
package reports

func GenerateTestReport(stats TransformStats, outputPath string) error {
    report := TestReport{
        Generated:    time.Now(),
        TotalEmails:  stats.TotalTransformed,
        ThreadCount:  stats.ThreadCount,
        PersonaCount: len(stats.PersonaMap),
        DateRange: DateRange{
            Start: /* calculate from messages */,
            End:   /* calculate from messages */,
        },
        TopContacts: getTopContacts(stats.PersonaMap),
        LabelDistribution: /* calculate */,
    }
    
    // Generate HTML report
    tmpl := template.Must(template.ParseFiles("test/reports/template.html"))
    // ... render template
}
```

### 7.2 View in DaProject UI

1. Start DaProject frontend in test mode:
```bash
cd web
REACT_APP_API_URL=http://localhost:8080 npm start
```

2. Login with test credentials
3. View the transformed Enron data as if it were your own email history
4. Test all visualizations and features with realistic data

## Troubleshooting

### Common Issues

1. **"Too many emails" error**: Reduce `--limit` parameter
2. **Memory issues**: Process in batches by date range
3. **Encoding errors**: The transformer handles most, but check `transform_stats.json` for skipped emails
4. **Missing personas**: Top 10 contacts are mapped to test personas, others become generic

### Performance Tips

- Use `kaminski-v` for balanced dataset (4,500 emails)
- Use `germany-c` for smaller tests (3,800 emails)
- Avoid `dasovich-j` unless testing scale (28,000 emails)
- Pre-generate and cache test data for CI/CD

## Advanced Usage

### Custom Personas

Edit the personas array in `assignPersonas()` to match your test scenarios:

```go
personas := []TestPersona{
    {Name: "Alice Johnson", Email: "alice@family.com", Role: "spouse"},
    {Name: "Bob Smith", Email: "bob@techcorp.com", Role: "cto"},
    // Add your custom personas
}
```

### Time Period Adjustment

Change the time shift in `NewGmailTransformer()`:

```go
// Shift to 1 year ago instead of 3
baseDate := time.Now().AddDate(-1, 0, 0)
```

### Filter Specific Content

Add filters in `LoadEnronEmails()`:

```go
// Only load emails with attachments
if !hasAttachments(email) {
    continue
}
```

---

This completes the setup for using Enron email data as realistic test fixtures for DaProject. The transformed data provides authentic communication patterns, relationship evolution, and timeline density for comprehensive integration testing.