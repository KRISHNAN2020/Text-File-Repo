using CashFlowCentralFlows;
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
using CashFlowCentralFlows;
using CashFlowCentralApiTests.Tests.AccountReceivables;
using EPay.Test.TestSubscriberRepositories.Models;
using static Fiserv.CashFlowCentralTesting.LddMicroservice.LddMicroserviceCaller;
using Fiserv.CashFlowCentralTesting.LddMicroservice;
using ReportPortal.Client.Abstractions.Models;
using static Fiserv.CashFlowCentralTesting.Models.Invoice.PostInvoiceRequest;
using static Fiserv.CashFlowCentralTesting.Models.Invoice.CustomerInfo;
using CashFlowCentralCosmosCallers;
using REMIT_MASTER = EPay.Test.DB.Genesis.REMIT_MASTER;
using Microsoft.Azure.Cosmos.Linq;
using static Fiserv.CashFlowCentralTesting.Models.Invoice.PutInvoiceRequest;

namespace CashFlowCentralApiTests.Tests.Invoice
{
    [Parallelizable(ParallelScope.Fixtures)]
    [FixtureLifeCycle(LifeCycle.SingleInstance)]
    [Ignore("Ignoring the entire test suite to identify the root cause of junk data in invoice details from Cosmos DB")]
    public class DeleteInvoiceTests
    {
        EPayTestContext epayTestContext;
        CashFlowCentralCaller cashFlowCentralCaller;
        CfcTestData currentTestData;
        string subscriberAuthToken;
        DataStore testDataStore;
        private ILogger _log = TestSetup.Log;
        private IConfiguration _config = TestSetup.Config;
        CashFlowCentralTestDataProvider? testDataProvider;
        InvoiceCosmosCaller invoiceCosmosCaller;

        [OneTimeSetUp]
        public void OneTimeSetup()
        {
            testDataStore = new CashFlowCentralTestDataProvider(_log, _config).TestDataStore;
        }

        [OneTimeTearDown]
        public void OneTimeTeardown()
        {
            testDataStore.Dispose();
        }

        [SetUp]
        public void Setup()
        {
            epayTestContext = new EPayTestContext(TestContext.CurrentContext, TestSetup.Log, TestContext.Parameters);
            cashFlowCentralCaller = new CashFlowCentralCaller(epayTestContext, TestSetup.Config);
            testDataProvider = new CashFlowCentralTestDataProvider(_log, _config);
            currentTestData = testDataProvider.ProvideTestData(epayTestContext)!;
            invoiceCosmosCaller = new InvoiceCosmosCaller(epayTestContext, TestSetup.Config);
        }

        public string GenerateAuthenticationToken(string sponsorId, string subscriberId, string businessUserId)
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
        }

        [TearDown]
        public void Teardown()
        {
            epayTestContext.Report.EndTest();
        }
        
        [Test]
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
            var postInvoiceRsp = cashFlowCentralCaller.PostInvoice(postInvoiceRequest, subscriberAuthToken);      epayTestContext.Report.AreEqual("Verify status", "Created", postInvoiceRsp.HttpResponse.StatusCode);
            var cosmosDbRes = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
            epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", "Active", cosmosDbRes.Result.FirstOrDefault().Status);
            var deleteInvoiceRsp = cashFlowCentralCaller.DeleteInvoice(postInvoiceRequest.invoiceId, subscriberAuthToken);            
            var cosmosDbRsp = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
            epayTestContext.Report.AreEqual("Verify Invoice ID inCosmos DB", postInvoiceRequest.invoiceId, cosmosDbRsp.Result.FirstOrDefault().InvoiceId);
            epayTestContext.Report.AreEqual("Verify Invoice Number in Cosmos DB", postInvoiceRequest.invoiceNumber, cosmosDbRsp.Result.FirstOrDefault().InvoiceNumber);
            epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", "Canceled", cosmosDbRsp.Result.FirstOrDefault().Status);            
        }

        [Test]
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
            var postInvoiceRsp = cashFlowCentralCaller.PostInvoice(postInvoiceRequest, subscriberAuthToken); epayTestContext.Report.AreEqual("Verify status", "Created", postInvoiceRsp.HttpResponse.StatusCode);
            var cosmosDbRsp = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
            epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", postInvoiceRequest.status, cosmosDbRsp.Result.FirstOrDefault().Status);
            var deleteInvoiceRsp = cashFlowCentralCaller.DeleteInvoice(postInvoiceRequest.invoiceId, subscriberAuthToken);            
            var cosmosDbRes = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
            epayTestContext.Report.AreEqual("Verify Invoice ID inCosmos DB", postInvoiceRequest.invoiceId, cosmosDbRes.Result.FirstOrDefault().InvoiceId);
            epayTestContext.Report.AreEqual("Verify Invoice Number in Cosmos DB", postInvoiceRequest.invoiceNumber, cosmosDbRes.Result.FirstOrDefault().InvoiceNumber);
            epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", "Canceled", cosmosDbRes.Result.FirstOrDefault().Status);
        }
       
        [Test]
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
            var postInvoiceRsp = cashFlowCentralCaller.PostInvoice(postInvoiceRequest, subscriberAuthToken); epayTestContext.Report.AreEqual("Verify status", "Created", postInvoiceRsp.HttpResponse.StatusCode); var deleteInvoiceRsp = cashFlowCentralCaller.DeleteInvoice(postInvoiceRequest.invoiceId, subscriberAuthToken);
            var cosmosDbRsp = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
            epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", "Canceled", cosmosDbRsp.Result.FirstOrDefault().Status);
            var deleteInvoiceRes = cashFlowCentralCaller.DeleteInvoice(postInvoiceRequest.invoiceId, subscriberAuthToken);
            epayTestContext.Report.AreEqual("Verify Error type", "/problems/invoice-alreadycanceled", deleteInvoiceRes.Type);
            epayTestContext.Report.AreEqual("Verify Error title", "Invoice already canceled", deleteInvoiceRes.Title);
            epayTestContext.Report.AreEqual("Verify Error status", "400", deleteInvoiceRes.Status);
            epayTestContext.Report.AreEqual("Verify Error detail", "Invoice was previously canceled.", deleteInvoiceRes.Detail);
            epayTestContext.Report.AreEqual("Verify Error instance", "/user/v1/self/arinvoices/" + postInvoiceRequest.invoiceId, deleteInvoiceRes.Instance);
        }
        
        [Test]
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
            var postInvoiceRsp = cashFlowCentralCaller.PostInvoice(postInvoiceRequest, subscriberAuthToken); epayTestContext.Report.AreEqual("Verify status", "Created", postInvoiceRsp.HttpResponse.StatusCode);

            var putInvoiceRequest = new PutInvoiceRequest().RandomPutInvoiceRequest();
            putInvoiceRequest.invoiceNumber = postInvoiceRequest.invoiceNumber;
            putInvoiceRequest.status = PutInvoiceStatus.Filed;
            putInvoiceRequest.customerInfo.type = CustomerType.Business;
            putInvoiceRequest.totalAmount = 1;
            var putInvoiceRes = cashFlowCentralCaller.PutInvoice(putInvoiceRequest, postInvoiceRequest.invoiceId, subscriberAuthToken);
            epayTestContext.Report.AreEqual("Verify Modify invoice success", "OK", putInvoiceRes.HttpResponse.StatusCode);

            var cosmosDbRsp = invoiceCosmosCaller.GetInvoiceDetails(postInvoiceRequest.invoiceId);
            epayTestContext.Report.AreEqual("Verify Invoice Status in Cosmos DB", "Filed", cosmosDbRsp.Result.FirstOrDefault().Status);
            var deleteInvoiceRes = cashFlowCentralCaller.DeleteInvoice(postInvoiceRequest.invoiceId, subscriberAuthToken);
            epayTestContext.Report.AreEqual("Verify Error type", "/problems/invoice-invalidstatus", deleteInvoiceRes.Type);
            epayTestContext.Report.AreEqual("Verify Error title", "Invalid invoice status", deleteInvoiceRes.Title);
            epayTestContext.Report.AreEqual("Verify Error status", "400", deleteInvoiceRes.Status);
            epayTestContext.Report.AreEqual("Verify Error detail", "The status of the invoice does not allow to be deleted.", deleteInvoiceRes.Detail);
            epayTestContext.Report.AreEqual("Verify Error instance", "/user/v1/self/arinvoices/"+ postInvoiceRequest.invoiceId, deleteInvoiceRes.Instance);
        }
        
        [Test]
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
        }

        [Test]
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
        }

        [Test]
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
        }
        //TTL Calculation
        [Test]
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
        }

        //AR Enrollment end-to-end flow pending added the existing data in json according to datarefresh for time being 
        private CfcTestData CreateBusinessWithARCapable(EPayTestContext epayTestContext, ILogger logger, IConfiguration config)
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
                // Patch LDD

                //Merchant Vetting update API
            }
        }

    }
}
