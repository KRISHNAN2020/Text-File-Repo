Let’s dive into this C# code for the DeleteInvoiceTests class, part of an API testing suite using NUnit. This class tests the invoice deletion functionality in a financial or invoicing system, likely called CashFlowCentral. I’ll walk you through every line of code, explaining what it does, why it’s there, and how it fits into the bigger picture. We’ll use markdown to keep things clear and organized.OverviewThe DeleteInvoiceTests class is a test suite written in C# using the NUnit framework. It’s designed to test the deletion of invoices under various conditions (e.g., different statuses like Active, Draft, Canceled, etc.) in an API integrated with a Cosmos DB backend. The code is located in the CashFlowCentralApiTests.Tests.Invoice namespace and is part of a larger testing framework for a financial application.Code Breakdown1. Namespace and Using Directivesusing CashFlowCentralFlows;
using EPay.Test.Utility;
using EPay.Test;
using Fiserv.CashFlowCentralTesting.Models;
using Fiserv.CashFlowCentralTesting.Utilities;
using Fiserv.CashFlowCentralTesting;
using JsonFlatFileDataStore;
using NUnit.Framework;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Configuration;
using Fiserv.CashFlowCentralTesting.Models.Transactions;
using Fiserv.CashFlowCentralTesting.Utilities.Attributes;
using System.Net;
using Fiserv.CashFlowCentralTesting.Models.Invoice;
using CashFlowCentralCosmosCallers;
using Bogus;
using Fiserv.CashFlowCentralTesting.Models.Enrollment;
using Fiserv.CashFlowCentralTesting.Models.Common;
using EPay.Test.DB.Genesis;
using System.Reflection;
using EPay.Test.DB;
using Fiserv.CashFlowCentralTesting.Models.ApplicationForm;
using CashFlowCentralApiTests.Tests.AccountReceivables;
using EPay.Test.TestSubscriberRepositories.Models;
using static Fiserv.CashFlowCentralTesting.LddMicroservice.LddMicroserviceCaller;
using Fiserv.CashFlowCentralTesting.LddMicroservice;
using ReportPortal.Client.Abstractions.Models;
using static Fiserv.CashFlowCentralTesting.Models.Invoice.PostInvoiceRequest;
using static Fiserv.CashFlowCentralTesting.Models.Invoice.CustomerInfo;
using REMIT_MASTER = EPay.Test.DB.Genesis.REMIT_MASTER;
using Microsoft.Azure.Cosmos.Linq;
using static Fiserv.CashFlowCentralTesting.Models.Invoice.PutInvoiceRequest;What’s Happening: These using statements import namespaces that provide classes, methods, and models used in the test class. For example:NUnit.Framework provides testing attributes like [Test].Fiserv.CashFlowCentralTesting.Models.Invoice includes invoice-related models (e.g., PostInvoiceRequest).CashFlowCentralCosmosCallers likely provides database access to Cosmos DB.Bogus is a library for generating fake data, useful for testing.using static (e.g., PostInvoiceRequest) allows direct access to static members without prefixing the class name.Why: This ensures all necessary tools and models are available, reducing the need to fully qualify names later.2. Namespace Declarationnamespace CashFlowCentralApiTests.Tests.Invoice {What’s Happening: Defines the namespace for this class, organizing it under CashFlowCentralApiTests.Tests.Invoice. This suggests it’s part of a broader test suite focused on invoice-related API endpoints.Why: Namespaces help avoid naming conflicts and group related code logically.3. Test Fixture Attributes[Parallelizable(ParallelScope.Fixtures)]
[FixtureLifeCycle(LifeCycle.SingleInstance)]
[Ignore("Ignoring the entire test suite to identify the root cause of junk data in invoice details from Cosmos DB")]
public class DeleteInvoiceTests {What’s Happening:[Parallelizable(ParallelScope.Fixtures)]: Allows this test fixture to run in parallel with other fixtures, speeding up test execution.[FixtureLifeCycle(LifeCycle.SingleInstance)]: Ensures only one instance of this class is created, reused across all tests in this suite.[Ignore("...")]: Skips the entire test suite with a note about “junk data” in Cosmos DB, indicating a known issue that needs fixing before these tests can run reliably.Why: These attributes control how NUnit executes and manages the tests. The [Ignore] suggests debugging or data issues are in progress.4. Class FieldsEPayTestContext epayTestContext;
CashFlowCentralCaller cashFlowCentralCaller;
CfcTestData currentTestData;
string subscriberAuthToken;
DataStore testDataStore;
private ILogger _log = TestSetup.Log;
private IConfiguration _config = TestSetup.Config;
CashFlowCentralTestDataProvider? testDataProvider;
InvoiceCosmosCaller invoiceCosmosCaller;What’s Happening: These are instance fields used across the test methods:epayTestContext: Manages test state and reporting.cashFlowCentralCaller: Makes API calls to the CashFlowCentral system.currentTestData: Holds test data (e.g., business details) for the current test.subscriberAuthToken: Stores the authentication token for API calls.testDataStore: A data store for test data, likely using JsonFlatFileDataStore._log: A logger from TestSetup for logging test events._config: Configuration settings from TestSetup.testDataProvider: Provides test data, nullable (?) to indicate it might not always be initialized.invoiceCosmosCaller: Interacts with Cosmos DB for invoice data.Why: These fields centralize resources needed by multiple tests, avoiding redundant setup.5. OneTimeSetUp Method[OneTimeSetUp]
public void OneTimeSetup()
{
    testDataStore = new CashFlowCentralTestDataProvider(_log, _config).TestDataStore;
}What’s Happening: Marked with [OneTimeSetUp], this runs once before any tests in the suite. It initializes testDataStore using a CashFlowCentralTestDataProvider.Why: Sets up shared resources (the data store) that persist across all tests, improving efficiency.6. OneTimeTearDown Method[OneTimeTearDown]
public void OneTimeTeardown()
{
    testDataStore.Dispose();
}What’s Happening: Marked with [OneTimeTearDown], this runs once after all tests finish. It disposes of testDataStore to free resources.Why: Ensures cleanup after testing, preventing resource leaks.7. Setup Method[SetUp]
public void Setup()
{
    epayTestContext = new EPayTestContext(TestContext.CurrentContext, TestSetup.Log, TestContext.Parameters);
    cashFlowCentralCaller = new CashFlowCentralCaller(epayTestContext, TestSetup.Config);
    testDataProvider = new CashFlowCentralTestDataProvider(_log, _config);
    currentTestData = testDataProvider.ProvideTestData(epayTestContext)!;
    invoiceCosmosCaller = new InvoiceCosmosCaller(epayTestContext, TestSetup.Config);
}What’s Happening: Marked with [SetUp], this runs before each test:Initializes epayTestContext with the current test context, logger, and parameters.Creates cashFlowCentralCaller for API calls.Sets up testDataProvider.Retrieves currentTestData (the ! asserts it’s not null).Initializes invoiceCosmosCaller for Cosmos DB interactions.Why: Prepares a fresh environment for each test, ensuring isolation.8. Authentication Token Generationpublic string GenerateAuthenticationToken(string sponsorId, string subscriberId, string businessUserId)
{
    using (var scope = ReportPortal.Shared.Context.Current.Log.BeginScope("**Authentication**"))
    {
        subscriberAuthToken = cashFlowCentralCaller.UserAuthGadget(sponsorId, subscriberId, businessUserId)?.AccessToken;

        if (string.IsNullOrWhiteSpace(subscriberAuthToken))
        {
            throw new Exception("SSO Login not successful");
        }
    }
    return subscriberAuthToken;
}What’s Happening:Takes sponsorId, subscriberId, and businessUserId as parameters.Uses cashFlowCentralCaller.UserAuthGadget to get an access token, storing it in subscriberAuthToken.Logs the process with ReportPortal.Throws an exception if the token is empty or null.Returns the token.Why: Provides a reusable way to authenticate API calls, critical for testing secured endpoints.9. Teardown Method[TearDown]
public void Teardown()
{
    epayTestContext.Report.EndTest();
}What’s Happening: Marked with [TearDown], this runs after each test to finalize reporting via epayTestContext.Why: Ensures test results are logged and reported correctly.10. Test MethodsLet’s examine each test method in detail. They follow a pattern: set up data, perform an action (delete an invoice), and verify the outcome.Test 1: DeleteInvoiceWithActiveStatus[Test]
[DataRefresh(RefreshMethodName = "CreateBusinessWithARCapable")]
public async Task DeleteInvoiceWithActiveStatus()
{
    var subscriberAuthToken = GenerateAuthenticationToken(currentTestData.Business.SponsorId, currentTestData.Business.SubscriberId, currentTestData.Business.BusinessUserId);
    string ckfr_sbsr_id = currentTestData.Business.CheckfreeSubscriberId;
    var sponsorId = currentTestData.Business.SponsorId;
    var postInvoiceRequest = new PostInvoiceRequest().RandomPostInvoiceRequest();
    postInvoiceRequest.status = InvoiceStatus.Active;
    postInvoiceRequest.customerInfo.type = CustomerType.Business;
    postInvoiceRequest.totalAmount = 1;
    var postInvoiceRsp = cashFlowCentralCaller.PostInvoice(postInvoiceRequest, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify status", "Created", postInvoiceRsp.HttpResponse.StatusCode);
    var cosmosDbRes = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
    epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", "Active", cosmosDbRes.Result.FirstOrDefault().Status);
    var deleteInvoiceRsp = cashFlowCentralCaller.DeleteInvoice(postInvoiceRequest.invoiceId, subscriberAuthToken);
    var cosmosDbRsp = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
    epayTestContext.Report.AreEqual("Verify Invoice ID inCosmos DB", postInvoiceRequest.invoiceId, cosmosDbRsp.Result.FirstOrDefault().InvoiceId);
    epayTestContext.Report.AreEqual("Verify Invoice Number in Cosmos DB", postInvoiceRequest.invoiceNumber, cosmosDbRsp.Result.FirstOrDefault().InvoiceNumber);
    epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", "Canceled", cosmosDbRsp.Result.FirstOrDefault().Status);
}What’s Happening:[Test]: Marks this as a test method.[DataRefresh("CreateBusinessWithARCapable")]: Refreshes test data using the named method.Gets an authentication token.Creates an Active invoice with a random PostInvoiceRequest.Posts the invoice and verifies the HTTP status is “Created”.Checks Cosmos DB to confirm the status is “Active”.Deletes the invoice and verifies in Cosmos DB that the status changes to “Canceled” and other details (ID, number) match.Why: Tests that an active invoice can be deleted, changing its status to “Canceled”.Test 2: DeleteInvoiceWithDraftStatus[Test]
[DataRefresh(RefreshMethodName = "CreateBusinessWithARCapable")]
public async Task DeleteInvoiceWithDraftStatus()
{
    var subscriberAuthToken = GenerateAuthenticationToken(currentTestData.Business.SponsorId, currentTestData.Business.SubscriberId, currentTestData.Business.BusinessUserId);
    string ckfr_sbsr_id = currentTestData.Business.CheckfreeSubscriberId;
    var sponsorId = currentTestData.Business.SponsorId;
    var postInvoiceRequest = new PostInvoiceRequest().RandomPostInvoiceRequest();
    postInvoiceRequest.status = InvoiceStatus.Draft;
    postInvoiceRequest.customerInfo.type = CustomerType.Business;
    postInvoiceRequest.totalAmount = 1;
    var postInvoiceRsp = cashFlowCentralCaller.PostInvoice(postInvoiceRequest, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify status", "Created", postInvoiceRsp.HttpResponse.StatusCode);
    var cosmosDbRsp = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
    epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", postInvoiceRequest.status, cosmosDbRsp.Result.FirstOrDefault().Status);
    var deleteInvoiceRsp = cashFlowCentralCaller.DeleteInvoice(postInvoiceRequest.invoiceId, subscriberAuthToken);
    var cosmosDbRes = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
    epayTestContext.Report.AreEqual("Verify Invoice ID inCosmos DB", postInvoiceRequest.invoiceId, cosmosDbRes.Result.FirstOrDefault().InvoiceId);
    epayTestContext.Report.AreEqual("Verify Invoice Number in Cosmos DB", postInvoiceRequest.invoiceNumber, cosmosDbRes.Result.FirstOrDefault().InvoiceNumber);
    epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", "Canceled", cosmosDbRes.Result.FirstOrDefault().Status);
}What’s Happening: Similar to the previous test, but starts with a Draft invoice. Verifies it’s created, then deleted, with the status updating to “Canceled”.Why: Ensures draft invoices can also be deleted successfully.Test 3: DeleteInvoiceWithCanceledStatus[Test]
[DataRefresh(RefreshMethodName = "CreateBusinessWithARCapable")]
public async Task DeleteInvoiceWithCanceledStatus()
{
    var subscriberAuthToken = GenerateAuthenticationToken(currentTestData.Business.SponsorId, currentTestData.Business.SubscriberId, currentTestData.Business.BusinessUserId);
    string ckfr_sbsr_id = currentTestData.Business.CheckfreeSubscriberId;
    var sponsorId = currentTestData.Business.SponsorId;
    var postInvoiceRequest = new PostInvoiceRequest().RandomPostInvoiceRequest();
    postInvoiceRequest.status = InvoiceStatus.Draft;
    postInvoiceRequest.customerInfo.type = CustomerType.Business;
    postInvoiceRequest.totalAmount = 1;
    var postInvoiceRsp = cashFlowCentralCaller.PostInvoice(postInvoiceRequest, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify status", "Created", postInvoiceRsp.HttpResponse.StatusCode);
    var deleteInvoiceRsp = cashFlowCentralCaller.DeleteInvoice(postInvoiceRequest.invoiceId, subscriberAuthToken);
    var cosmosDbRsp = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
    epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", "Canceled", cosmosDbRsp.Result.FirstOrDefault().Status);
    var deleteInvoiceRes = cashFlowCentralCaller.DeleteInvoice(postInvoiceRequest.invoiceId, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify Error type", "/problems/invoice-alreadycanceled", deleteInvoiceRes.Type);
    epayTestContext.Report.AreEqual("Verify Error title", "Invoice already canceled", deleteInvoiceRes.Title);
    epayTestContext.Report.AreEqual("Verify Error status", "400", deleteInvoiceRes.Status);
    epayTestContext.Report.AreEqual("Verify Error detail", "Invoice was previously canceled.", deleteInvoiceRes.Detail);
    epayTestContext.Report.AreEqual("Verify Error instance", "/user/v1/self/arinvoices/" + postInvoiceRequest.invoiceId, deleteInvoiceRes.Instance);
}What’s Happening: Creates a draft invoice, deletes it (status becomes “Canceled”), then tries to delete it again. Expects a 400 error with specific error details.Why: Verifies the API prevents deleting an already canceled invoice.Test 4: DeleteInvoiceWithFiledStatus[Test]
[DataRefresh(RefreshMethodName = "CreateBusinessWithARCapable")]
public async Task DeleteInvoiceWithFiledStatus()
{
    var subscriberAuthToken = GenerateAuthenticationToken(currentTestData.Business.SponsorId, currentTestData.Business.SubscriberId, currentTestData.Business.BusinessUserId);
    string ckfr_sbsr_id = currentTestData.Business.CheckfreeSubscriberId;
    var sponsorId = currentTestData.Business.SponsorId;
    var postInvoiceRequest = new PostInvoiceRequest().RandomPostInvoiceRequest();
    postInvoiceRequest.status = InvoiceStatus.Draft;
    postInvoiceRequest.customerInfo.type = CustomerType.Business;
    postInvoiceRequest.totalAmount = 1;
    var postInvoiceRsp = cashFlowCentralCaller.PostInvoice(postInvoiceRequest, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify status", "Created", postInvoiceRsp.HttpResponse.StatusCode);

    var putInvoiceRequest = new PutInvoiceRequest().RandomPutInvoiceRequest();
    putInvoiceRequest.invoiceNumber = postInvoiceRequest.invoiceNumber;
    putInvoiceRequest.status = PutInvoiceStatus.Filed;
    putInvoiceRequest.customerInfo.type = CustomerType.Business;
    putInvoiceRequest.totalAmount = 1;
    var putInvoiceRes = cashFlowCentralCaller.PutInvoice(putInvoiceRequest, postInvoiceRequest.invoiceId, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify Modify invoice success", "OK", putInvoiceRes.HttpResponse.StatusCode);

    var cosmosDbRsp = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
    epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos SOUND", "Filed", cosmosDbRsp.Result.FirstOrDefault().Status);
    var deleteInvoiceRes = cashFlowCentralCaller.DeleteInvoice(postInvoiceRequest.invoiceId, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify Error type", "/problems/invoice-invalidstatus", deleteInvoiceRes.Type);
    epayTestContext.Report.AreEqual("Verify Error title", "Invalid invoice status", deleteInvoiceRes.Title);
    epayTestContext.Report.AreEqual("Verify Error status", "400", deleteInvoiceRes.Status);
    epayTestContext.Report.AreEqual("Verify Error detail", "The status of the invoice does not allow to be deleted.", deleteInvoiceRes.Detail);
    epayTestContext.Report.AreEqual("Verify Error instance", "/user/v1/self/arinvoices/"+ postInvoiceRequest.invoiceId, deleteInvoiceRes.Instance);
}What’s Happening: Creates a draft invoice, updates it to “Filed” using a PutInvoiceRequest, then tries to delete it. Expects a 400 error.Why: Ensures filed invoices cannot be deleted, testing status restrictions.Test 5: DeleteInvoiceWithPaidStatus[Test]
[DataRefresh(RefreshMethodName = "CreateBusinessWithARCapable")]
public async Task DeleteInvoiceWithPaidStatus()
{
    var subscriberAuthToken = GenerateAuthenticationToken(currentTestData.Business.SponsorId, currentTestData.Business.SubscriberId, currentTestData.Business.BusinessUserId);
    var invoiceId = "GuestInvoice139";
    var getInvoiceRsp = cashFlowCentralCaller.GetInvoiceByIDUserRoute(invoiceId, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify invoice status", "Paid", getInvoiceRsp.data.Status);
    var deleteInvoiceRes = cashFlowCentralCaller.DeleteInvoice(invoiceId, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify Error type", "/problems/invoice-invalidstatus", deleteInvoiceRes.Type);
    epayTestContext.Report.AreEqual("Verify Error title", "Invalid invoice status", deleteInvoiceRes.Title);
    epayTestContext.Report.AreEqual("Verify Error status", "400", deleteInvoiceRes.Status);
    epayTestContext.Report.AreEqual("Verify Error detail", "The status of the invoice does not allow to be deleted.", deleteInvoiceRes.Detail);
    epayTestContext.Report.AreEqual("Verify Error instance", "/user/v1/self/arinvoices/" + invoiceId, deleteInvoiceRes.Instance);
}What’s Happening: Uses a hardcoded “Paid” invoice ID, verifies its status, and attempts deletion. Expects a 400 error.Why: Tests that paid invoices are protected from deletion.Test 6: DeleteInvoiceWithFailedStatus[Test]
[DataRefresh(RefreshMethodName = "CreateBusinessWithARCapable")]
public async Task DeleteInvoiceWithFailedStatus()
{
    var subscriberAuthToken = GenerateAuthenticationToken(currentTestData.Business.SponsorId, currentTestData.Business.SubscriberId, currentTestData.Business.BusinessUserId);
    var invoiceId = "GuestInvoice167";
    var getInvoiceRsp = cashFlowCentralCaller.GetInvoiceByIDUserRoute(invoiceId, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify invoice status", "Expired", getInvoiceRsp.data.Status);
    var deleteInvoiceRes = cashFlowCentralCaller.DeleteInvoice(invoiceId, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify Error type", "/problems/invoice-invalidstatus", deleteInvoiceRes.Type);
    epayTestContext.Report.AreEqual("Verify Error title", "Invalid invoice status", deleteInvoiceRes.Title);
    epayTestContext.Report.AreEqual("Verify Error status", "400", deleteInvoiceRes.Status);
    epayTestContext.Report.AreEqual("Verify Error detail", "The status of the invoice does not allow to be deleted.", deleteInvoiceRes.Detail);
    epayTestContext.Report.AreEqual("Verify Error instance", "/user/v1/self/arinvoices/" + invoiceId, deleteInvoiceRes.Instance);
}What’s Happening: Tests an “Expired” (assumed failed) invoice, expecting a 400 error on deletion.Why: Verifies expired/failed invoices cannot be deleted.Test 7: DeleteInvoiceWithPendingStatus[Test]
[DataRefresh(RefreshMethodName = "CreateBusinessWithARCapable")]
public async Task DeleteInvoiceWithPendingStatus()
{
    var subscriberAuthToken = GenerateAuthenticationToken(currentTestData.Business.SponsorId, currentTestData.Business.SubscriberId, currentTestData.Business.BusinessUserId);
    var invoiceId = "InvoiceDetail12323";
    var getInvoiceRsp = cashFlowCentralCaller.GetInvoiceByIDUserRoute(invoiceId, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify invoice status", "Pending", getInvoiceRsp.data.Status);
    var deleteInvoiceRes = cashFlowCentralCaller.DeleteInvoice(invoiceId, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify Error type", "/problems/invoice-invalidstatus", deleteInvoiceRes.Type);
    epayTestContext.Report.AreEqual("Verify Error title", "Invalid invoice status", deleteInvoiceRes.Title);
    epayTestContext.Report.AreEqual("Verify Error status", "400", deleteInvoiceRes.Status);
    epayTestContext.Report.AreEqual("Verify Error detail", "The status of the invoice does not allow to be deleted.", deleteInvoiceRes.Detail);
    epayTestContext.Report.AreEqual("Verify Error instance", "/user/v1/self/arinvoices/" + invoiceId, deleteInvoiceRes.Instance);
}What’s Happening: Tests a “Pending” invoice, expecting a 400 error on deletion.Why: Ensures pending invoices are protected.Test 8: DeleteInvoiceWithInvalidInvoiceId[Test]
[DataRefresh(RefreshMethodName = "CreateBusinessWithARCapable")]
public async Task DeleteInvoiceWithInvalidInvoiceId()
{
    var subscriberAuthToken = GenerateAuthenticationToken(currentTestData.Business.SponsorId, currentTestData.Business.SubscriberId, currentTestData.Business.BusinessUserId);
    var invoiceId = "Invalid";
    var deleteInvoiceRes = cashFlowCentralCaller.DeleteInvoice(invoiceId, subscriberAuthToken);
    epayTestContext.Report.AreEqual("Verify Error type", "/problems/invoice-notfound", deleteInvoiceRes.Type);
    epayTestContext.Report.AreEqual("Verify Error title", "Invoice not found", deleteInvoiceRes.Title);
    epayTestContext.Report.AreEqual("Verify Error status", "400", deleteInvoiceRes.Status);
    epayTestContext.Report.AreEqual("Verify Error detail", "Invoice not found.", deleteInvoiceRes.Detail);
    epayTestContext.Report.AreEqual("Verify Error instance", "/user/v1/self/arinvoices/" + invoiceId, deleteInvoiceRes.Instance);
}What’s Happening: Tries to delete an invalid invoice ID, expecting a 400 “not found” error.Why: Tests error handling for nonexistent invoices.11. Data Refresh Methodprivate CfcTestData CreateBusinessWithARCapable(EPayTestContext epayTestContext, ILogger logger, IConfiguration config)
{
    using (var scope = ReportPortal.Shared.Context.Current.Log.BeginScope($"**Data creation method: {MethodBase.GetCurrentMethod().Name}**"))
    {
        var faker = new Faker("en");
        string sponsorId = "12856";
        cashFlowCentralCaller = new CashFlowCentralCaller(epayTestContext, config);
        EnrollmentFlows enrollmentFlow = new EnrollmentFlows(epayTestContext, config);
        var testBusiness = enrollmentFlow.CreateRandomTestBusiness(sponsorId);
        CfcTestData cfcTestData = new CfcTestData { Business = testBusiness };
        enrollmentFlow.UpdateBusinessToAPARPro(cfcTestData.Business);
        subscriberAuthToken = cashFlowCentralCaller.UserAuthGadget(testBusiness.SponsorId, testBusiness.SubscriberId, testBusiness.BusinessUserId)?.AccessToken;
        ApplicationFormFlows applicationFlow = new ApplicationFormFlows(epayTestContext, config);
        var postApplicationFormRequest = new PostApplicationFormRequest().RandomPostApplicationFormRequest();
        postApplicationFormRequest.DebitBankAccountId = testBusiness.BankAccount.CommonId;
        postApplicationFormRequest.CreditBankAccountId = testBusiness.BankAccount.CommonId;
        var postApplication = cashFlowCentralCaller.PostApplicationForm(postApplicationFormRequest, subscriberAuthToken);
        return cfcTestData;
    }
}What’s Happening:Creates a test business with accounts receivable (AR) capabilities.Uses Faker for fake data, sets a sponsor ID, and enrolls a random business.Updates it to AP/AR Pro status, generates a token, and submits an application form.Returns the test data.Why: Provides fresh, AR-capable business data for each test, referenced by [DataRefresh].SummaryThe DeleteInvoiceTests class rigorously tests the invoice deletion API endpoint:Success Cases: Invoices in Active and Draft statuses can be deleted, updating to “Canceled”.Error Cases: Invoices in Canceled, Filed, Paid, Expired, Pending, or invalid IDs cannot be deleted, returning appropriate 400 errors.Setup: Uses authentication, API calls, and Cosmos DB checks to ensure end-to-end functionality.Current State: The suite is ignored due to junk data issues in Cosmos DB, indicating it’s under maintenance.This code teaches you how to structure API tests, handle authentication, interact with a database, and verify both success and failure scenarios in a financial system. Let me know if you’d like deeper dives into any part!