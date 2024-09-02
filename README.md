# Säkerhet

_Säkerhet_ - from Swedish, meaning security, certainty

Helpful abstractions to ease the creation of integration tests in Go with some key ideas:

- Tests should be easy to **read, write, run and maintain**
- Using Docker containers with real instances of services (PostgreSQL, NGINX, GCP Pub/Sub, etc) with testcontainers
- Integrate with the standard Go library
- Encourage loose coupling between parts

Tests are a great source of documentation and great way of getting to know a project.

As the business requirement or documentation changes, the test case will also adjust according to the needs. This will make the test case good documentation for the developer and will increase the confidence of the developer when refactoring or doing something.

![safety-net](https://user-images.githubusercontent.com/42377845/198402912-d9cf2925-6a7b-4f5c-9709-e1f24a9f827b.jpg)

See an example integration test here below. This tests a happy flow that a correct request body will return a 200 HTTP code and the correct data is in response body, published to Google Pub/Sub and persisted to PostgreSQL.

``` go
package service_test

// some imports ommitted for privacy and simplicity
import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"testing"
	"time"

	"github.com/hashicorp/go-retryablehttp"
	mhs "***"
	mhsMocks "***"

	abstractedcontainers "github.com/averageflow/sakerhet/pkg/abstracted_containers"
	"github.com/averageflow/sakerhet/pkg/sakerhet"
	"***"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/stretchr/testify/mock"
	"github.com/stretchr/testify/suite"
)

/* Start test boilerplate */

type ServiceIntegrationTestSuite struct {
	suite.Suite
	TestContext         context.Context
	TestContextCancel   context.CancelFunc
	GCPPubSubContainer  *abstractedcontainers.GCPPubSubContainer
	Sakerhet            sakerhet.Sakerhet
	PostgreSQLContainer *abstractedcontainers.PostgreSQLContainer
	DBPool              *pgxpool.Pool
	// some legacy system that we mock here
	MockedMHS           *mhsMocks.MHSI
}

// Before suite starts
func (suite *ServiceIntegrationTestSuite) SetupSuite() {
	ctx := context.Background()

	// smart constructors take sane defaults with empty structs
	// customize if needed
	suite.Sakerhet = sakerhet.NewSakerhetIntegrationTest(sakerhet.SakerhetBuilder{
		PostgreSQL: &sakerhet.PostgreSQLIntegrationTestParams{},
		GCPPubSub:  &sakerhet.GCPPubSubIntegrationTestParams{},
	})

	// Spin up one PostgreSQL container for all the tests in the suite
	postgreSQLC, err := suite.Sakerhet.PostgreSQLIntegrationTester.ContainerStart(ctx)
	suite.Assert().Nil(err)

	suite.PostgreSQLContainer = postgreSQLC

	// Create DB pool that will be reused across tests
	dbpool, err := pgxpool.New(ctx, suite.PostgreSQLContainer.ConnectionURL)
	if err != nil {
		suite.T().Fatal(fmt.Errorf("Unable to create connection pool: %v\n", err))
	}

	suite.DBPool = dbpool

	// Setup schema that will be reused across tests
	initialSchema := []string{
		`
		CREATE TABLE events (
			uuid VARCHAR(100) NOT NULL PRIMARY KEY,
			ttyp INTEGER NOT NULL,
			blob BYTEA NOT NULL,
			bu_code VARCHAR,
			bu_type VARCHAR
    );
		`,
	}

	err = suite.Sakerhet.PostgreSQLIntegrationTester.InitSchema(ctx, suite.DBPool, initialSchema)
	suite.Assert().Nil(err)

	// Spin up one GCP Pub/Sub container for all the tests in the suite
	pubSubContainer, err := suite.Sakerhet.GCPPubSubIntegrationTester.ContainerStart(ctx)
	suite.Assert().Nil(err)

	suite.GCPPubSubContainer = pubSubContainer

	suite.MockedMHS = mhsMocks.NewMHSI(suite.T())
}

// Before each test
func (suite *ServiceIntegrationTestSuite) SetupTest() {
	ctx, cancel := context.WithTimeout(context.Background(), sakerhet.GetIntegrationTestTimeout())
	suite.TestContext = ctx
	suite.TestContextCancel = cancel
}

// After each test
func (suite *ServiceIntegrationTestSuite) TearDownTest() {
	if err := suite.Sakerhet.PostgreSQLIntegrationTester.TruncateTable(
		context.Background(),
		suite.DBPool,
		[]string{"events"},
	); err != nil {
		suite.T().Fatal(err)
	}
}

// After suite ends
func (suite *ServiceIntegrationTestSuite) TearDownSuite() {
	suite.DBPool.Close()
	_ = suite.PostgreSQLContainer.Terminate(context.Background())
	_ = suite.GCPPubSubContainer.Terminate(context.Background())
}

// Start the test suite if we are running integration tests
func TestStockServiceTestSuite(t *testing.T) {
	sakerhet.SkipIntegrationTestsWhenUnitTesting(t)
	suite.Run(t, new(ServiceIntegrationTestSuite))
}

// Create GCP Pub/Sub publisher type that satisfies the business logic service's contract
type gcpPubSubPublisher struct {
	gcpTester *sakerhet.GCPPubSubIntegrationTester
}

// Implement GCP Pub/Sub publisher logic with integration test container
func (p *gcpPubSubPublisher) Publish(ctx context.Context, event *database.Event) (*pubsub.PublishResult, error) {
	payload, err := json.Marshal(event)
	if err != nil {
		return nil, err
	}

	if err := p.gcpTester.PublishData(ctx, payload); err != nil {
		return nil, err
	}

	return &pubsub.PublishResult{ID: "dummy-pubsub-id"}, nil
}

// Prepare the business logic service with injected dependencies
func prepareStockServiceAPI(suite *ServiceIntegrationTestSuite) *api.Handler {
	db, err := database.New(&database.Config{
		Driver:   "postgres",
		Username: suite.Sakerhet.PostgreSQLIntegrationTester.User,
		Password: suite.Sakerhet.PostgreSQLIntegrationTester.Password,
		Host:     fmt.Sprintf("%s:%s", suite.PostgreSQLContainer.Host, suite.PostgreSQLContainer.MappedPort),
		Name:     suite.Sakerhet.PostgreSQLIntegrationTester.DB,
	})

	suite.Assert().Nil(err)

	stockService := service.New(
		db,
		&gcpPubSubPublisher{
			gcpTester: suite.Sakerhet.GCPPubSubIntegrationTester,
		},
		suite.MockedMHS,
	)

	stockServiceAPI, err := api.New(&api.Config{
		Port:     ":8765",
		Services: []api.ServerInterface{stockService},
	})
	suite.Assert().Nil(err)

	return stockServiceAPI
}

/* End test boilerplate */

func (suite *ServiceIntegrationTestSuite) TestHappyFlowCorrectRequestBodyReturns200SendToMHS() {
	stockServiceAPI := prepareStockServiceAPI(suite)

	go func() {
		if err := stockServiceAPI.Run(suite.TestContext); err != nil {
			log.Fatal(err)
		}
	}()

	defer func() {
		_ = stockServiceAPI.Shutdown()
	}()

	now := time.Now()
	transaction := api.TransactionType{}

	qty := float64(5)
	err := transaction.FromTtyp431(api.Ttyp431{
		EventSource: api.EventSource{
			EventId:   "1",
			System:    "MHS",
			Timestamp: now,
		},
		Item: api.Item{
			Number:        "12345678",
			Type:          "ART",
			UnitOfMeasure: "PIECES",
		},
		Quantity: &qty,
	})
	suite.Assert().Nil(err)
	sampleHTTPPayload := api.SimTransactionRequest{
		Transactions: []api.TransactionType{
			transaction,
		},
	}

	marshalledHTTPPayload, err := json.Marshal(sampleHTTPPayload)
	suite.Assert().Nil(err)

	// HTTP checks
	httpSituation := &sakerhet.HttpIntegrationTestSituation{
		Request: &sakerhet.HttpIntegrationTestSituationRequest{
			RequestURL:    "http://localhost:8765/v5/api/stock/STO/999",
			RequestMethod: http.MethodPost,
			RequestHeaders: []sakerhet.HttpHeaderValuePair{
				{Header: "Content-Type", Value: "application/json"},
				{Header: api.HttpHeaderClientID, Value: "IntegrationTestClient"},
				{Header: api.HttpHeaderSource, Value: "MHS"},
			},
			RequestBody: marshalledHTTPPayload,
		},
		Expectation: &sakerhet.HttpIntegrationTestSituationExpectation{
			ResponseStatusCode: http.StatusOK,
		},
		Timeout: 2 * time.Second,
	}

	// ensure mock has required interactions
	suite.MockedMHS.EXPECT().ShouldBeSentToMHS("IntegrationTestClient").Return(true).Once()
	suite.MockedMHS.EXPECT().NewHTTPClient().Return(retryablehttp.NewClient()).Once()

	suite.MockedMHS.EXPECT().
		CreateReservation(mock.AnythingOfType("*gin.Context"), mock.AnythingOfType("*retryablehttp.Client"), sampleHTTPPayload, api.BusinessUnit{Type: "STO", Code: "00999"}).
		Return(mhs.Response{
			Errors:   []api.StockError{},
			Data:     []api.TransactionType{sampleHTTPPayload.Transactions[0]},
			HttpCode: http.StatusOK,
		}, nil).Once()

	httpSituationResult, err := httpSituation.SituationChecker()
	suite.Assert().Nil(err)

	var httpSituationResultData api.SimTransactionResponse

	err = json.Unmarshal(httpSituationResult.ResponseBody, &httpSituationResultData)
	suite.Assert().Nil(err)

	suite.Assert().Empty(httpSituationResultData.Errors)

	suite.Assert().Len(httpSituationResultData.Data, 1)

	ttypResult, err := httpSituationResultData.Data[0].AsTtyp431()
	suite.Assert().Nil(err)
	suite.Assert().Equal("431", ttypResult.Ttyp)
	suite.Assert().Equal("MHS", ttypResult.EventSource.System)
	suite.Assert().Equal(api.ART, ttypResult.Item.Type)
	suite.Assert().Equal("12345678", ttypResult.Item.Number)

	suite.Assert().NotEmpty(httpSituationResultData.Id)

	// PubSub checks
	pubSubMessages, err := suite.Sakerhet.GCPPubSubIntegrationTester.ReadMessages(
		suite.TestContext,
	)
	suite.Assert().Nil(err)

	suite.Assert().Empty(pubSubMessages)

	// PostgreSQL checks
	rs, err := suite.Sakerhet.PostgreSQLIntegrationTester.FetchData(
		suite.TestContext,
		suite.DBPool,
		`SELECT uuid, ttyp, blob, bu_type, bu_code FROM events;`,
		readDBStockEventRowHandler,
	)
	if err != nil {
		suite.T().Fatal(fmt.Errorf("Error occured while fetching data from PostgreSQL: %w", err))
	}

	suite.Assert().Empty(rs)
}
```

## Docs

### Test separation

This is not a strict requirement but a good practice. We encourage you to keep the 2 types of test separated.

There are plenty of reasons developers might want to separate tests. Some simple examples might be:

- Integration tests are often slower, so you may want to only run them after the unit test (which are often much faster) have passed.
- Smoke tests that are run against the live application, generally after a deployment.
- Deploying the same app to different tenants.

Of course, this is by no means an exhaustive list!

#### Unit tests

Files containing unit test should be named as `*_test.go`.

Unit tests will not run if the special environment variable `SAKERHET_RUN_INTEGRATION_TESTS` is found.

#### Integration tests

Files containing integration tests should be named as `*_integration_test.go`.

Integration tests will be run if the special environment variable `SAKERHET_RUN_INTEGRATION_TESTS` is found.

Integration tests can then be run with `export SAKERHET_RUN_INTEGRATION_TESTS=Y; go test` or equivalent.

This allows a clean separation of test runs. Take a look at the [Makefile](Makefile) in the root of the repo for examples.

There is another special variable, `SAKERHET_INTEGRATION_TEST_TIMEOUT` which configures the timeout of each integration test in seconds. This variable defaults to `60`.
