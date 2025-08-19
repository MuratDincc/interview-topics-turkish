# CI/CD Pipelines

## Giriş

CI/CD (Continuous Integration/Continuous Deployment) pipelines, modern software development'da code quality, automated testing ve reliable deployment için temel araçlardır. Mid-level geliştiriciler için CI/CD pipeline'larını tasarlamak ve implement etmek, software delivery süreçlerini optimize etmede kritiktir. Bu dosya, pipeline design, automation strategies, testing integration ve deployment strategies'leri kapsar.

## CI/CD Pipeline Design

### 1. Pipeline Architecture
Multi-stage CI/CD pipeline tasarımı.

```yaml
# Azure DevOps Pipeline - Multi-stage CI/CD
trigger:
  branches:
    include:
    - main
    - develop
    - feature/*

variables:
  solution: '**/*.sln'
  buildPlatform: 'Any CPU'
  buildConfiguration: 'Release'
  dotNetVersion: '8.0.x'
  testResultsFormat: 'VSTest'
  testResultsPublishTarget: 'TestResults'

stages:
- stage: Build
  displayName: 'Build and Test'
  jobs:
  - job: Build
    displayName: 'Build .NET Application'
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: UseDotNet@2
      displayName: 'Use .NET 8.0'
      inputs:
        version: '$(dotNetVersion)'
        
    - script: |
        dotnet --version
        dotnet --list-sdks
      displayName: 'Display .NET Version'
      
    - task: DotNetCoreCLI@2
      displayName: 'Restore NuGet packages'
      inputs:
        command: 'restore'
        projects: '$(solution)'
        feedsToUse: 'select'
        
    - task: DotNetCoreCLI@2
      displayName: 'Build solution'
      inputs:
        command: 'build'
        projects: '$(solution)'
        arguments: '--configuration $(buildConfiguration) --no-restore'
        
    - task: DotNetCoreCLI@2
      displayName: 'Run unit tests'
      inputs:
        command: 'test'
        projects: '**/*Tests/*.csproj'
        arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage" --results-directory $(Agent.TempDirectory)/TestResults'
        publishTestResults: true
        
    - task: PublishCodeCoverageResults@1
      displayName: 'Publish code coverage'
      inputs:
        codeCoverageTool: 'Cobertura'
        summaryFileLocation: '$(Agent.TempDirectory)/TestResults/*/coverage.cobertura.xml'
        
    - task: DotNetCoreCLI@2
      displayName: 'Publish application'
      inputs:
        command: 'publish'
        projects: '**/MyApp/*.csproj'
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/publish --no-build'
        publishWebProjects: true
        
    - task: PublishBuildArtifacts@1
      displayName: 'Publish build artifacts'
      inputs:
        pathToPublish: '$(Build.ArtifactStagingDirectory)/publish'
        artifactName: 'drop'
        
    - task: PublishTestResults@2
      displayName: 'Publish test results'
      inputs:
        testResultsFormat: '$(testResultsFormat)'
        testResultsFiles: '**/TestResults/*.trx'
        testRunTitle: '$(Build.DefinitionName) - $(Build.BuildNumber)'
        failTaskOnFailedTests: true
        mergeTestResults: true

- stage: Security
  displayName: 'Security Analysis'
  dependsOn: Build
  condition: succeeded()
  jobs:
  - job: SecurityScan
    displayName: 'Security Scanning'
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: UseDotNet@2
      inputs:
        version: '$(dotNetVersion)'
        
    - script: |
        dotnet tool install --global dotnet-security-scan
        dotnet security-scan --project-path $(Build.SourcesDirectory)
      displayName: 'Run security scan'
      
    - script: |
        dotnet tool install --global dotnet-outdated-tool
        dotnet outdated --project-path $(Build.SourcesDirectory) --fail-on-updates
      displayName: 'Check for outdated packages'
      
    - task: SonarQubePrepare@4
      displayName: 'Prepare SonarQube analysis'
      inputs:
        SonarQube: 'SonarQube'
        scannerMode: 'CLI'
        configMode: 'manual'
        cliProjectKey: 'myapp'
        cliProjectName: 'MyApp'
        cliSources: '$(Build.SourcesDirectory)'
        
    - task: SonarQubeAnalyze@4
      displayName: 'Run SonarQube analysis'
      
    - task: SonarQubePublish@4
      displayName: 'Publish SonarQube results'
      inputs:
        pollingTimeoutSec: '300'

- stage: Quality
  displayName: 'Code Quality'
  dependsOn: Security
  condition: succeeded()
  jobs:
  - job: CodeQuality
    displayName: 'Code Quality Analysis'
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: UseDotNet@2
      inputs:
        version: '$(dotNetVersion)'
        
    - script: |
        dotnet tool install --global dotnet-format
        dotnet format --verify-no-changes --verbosity diagnostic
      displayName: 'Check code formatting'
      
    - script: |
        dotnet tool install --global dotnet-ef
        dotnet ef migrations list --project $(Build.SourcesDirectory)/src/MyApp.Infrastructure
      displayName: 'Check database migrations'
      
    - task: PublishBuildArtifacts@1
      displayName: 'Publish quality report'
      inputs:
        pathToPublish: '$(Build.ArtifactStagingDirectory)'
        artifactName: 'quality-report'

- stage: DeployStaging
  displayName: 'Deploy to Staging'
  dependsOn: Quality
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployStaging
    displayName: 'Deploy to Staging Environment'
    environment: 'staging'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: drop
            
          - task: AzureWebApp@1
            displayName: 'Deploy to Azure Web App (Staging)'
            inputs:
              azureSubscription: 'Azure Subscription'
              appName: 'myapp-staging'
              package: '$(Pipeline.Workspace)/drop/**/*.zip'
              appType: 'webApp'
              
          - script: |
              # Run smoke tests
              curl -f https://myapp-staging.azurewebsites.net/health
            displayName: 'Smoke test'
            
          - script: |
              # Run integration tests
              dotnet test --filter Category=Integration --logger trx --results-directory $(Agent.TempDirectory)/IntegrationTestResults
            displayName: 'Integration tests'
            
          - task: PublishTestResults@2
            displayName: 'Publish integration test results'
            inputs:
              testResultsFormat: 'VSTest'
              testResultsFiles: '$(Agent.TempDirectory)/IntegrationTestResults/*.trx'

- stage: DeployProduction
  displayName: 'Deploy to Production'
  dependsOn: DeployStaging
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployProduction
    displayName: 'Deploy to Production Environment'
    environment: 'production'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: drop
            
          - task: AzureWebApp@1
            displayName: 'Deploy to Azure Web App (Production)'
            inputs:
              azureSubscription: 'Azure Subscription'
              appName: 'myapp-production'
              package: '$(Pipeline.Workspace)/drop/**/*.zip'
              appType: 'webApp'
              
          - script: |
              # Verify deployment
              curl -f https://myapp-production.azurewebsites.net/health
            displayName: 'Health check'
            
          - task: PostDeploymentTest@1
            displayName: 'Post-deployment tests'
            inputs:
              testScript: '$(Build.SourcesDirectory)/scripts/post-deployment-tests.ps1'
              testResultsFile: '$(Agent.TempDirectory)/PostDeploymentTestResults.xml'
```

### 2. GitHub Actions Workflow
GitHub Actions ile CI/CD pipeline.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop, feature/* ]
  pull_request:
    branches: [ main, develop ]

env:
  DOTNET_VERSION: '8.0.x'
  SOLUTION_FILE: '**/*.sln'
  TEST_PATTERN: '**/*Tests/*.csproj'
  MAIN_PROJECT: '**/MyApp/*.csproj'

jobs:
  # Build and Test Job
  build-and-test:
    name: Build and Test
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
        
    - name: Restore dependencies
      run: dotnet restore ${{ env.SOLUTION_FILE }}
      
    - name: Build solution
      run: dotnet build ${{ env.SOLUTION_FILE }} --configuration Release --no-restore
      
    - name: Run unit tests
      run: |
        dotnet test ${{ env.TEST_PATTERN }} \
          --configuration Release \
          --no-build \
          --collect:"XPlat Code Coverage" \
          --results-directory $(pwd)/TestResults \
          --logger trx
          
    - name: Publish test results
      uses: actions/upload-artifact@v4
      with:
        name: test-results
        path: TestResults/
        
    - name: Generate coverage report
      run: |
        dotnet tool install --global dotnet-reportgenerator-globaltool
        reportgenerator \
          -reports:TestResults/*/coverage.cobertura.xml \
          -targetdir:coverage-report \
          -reporttypes:Html
          
    - name: Publish coverage report
      uses: actions/upload-artifact@v4
      with:
        name: coverage-report
        path: coverage-report/
        
    - name: Publish application
      run: |
        dotnet publish ${{ env.MAIN_PROJECT }} \
          --configuration Release \
          --output $(pwd)/publish \
          --no-build
          
    - name: Upload build artifacts
      uses: actions/upload-artifact@v4
      with:
        name: build-artifacts
        path: publish/

  # Security Analysis Job
  security-analysis:
    name: Security Analysis
    runs-on: ubuntu-latest
    needs: build-and-test
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
        
    - name: Run security scan
      run: |
        dotnet tool install --global dotnet-security-scan
        dotnet security-scan --project-path .
        
    - name: Check for vulnerabilities
      run: |
        dotnet tool install --global dotnet-outdated-tool
        dotnet outdated --project-path . --fail-on-updates
        
    - name: Run SonarQube analysis
      uses: sonarqube-quality-gate-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      with:
        args: >
          -Dsonar.projectKey=myapp
          -Dsonar.projectName=MyApp
          -Dsonar.sources=.
          -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }}

  # Code Quality Job
  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    needs: build-and-test
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
        
    - name: Check code formatting
      run: |
        dotnet tool install --global dotnet-format
        dotnet format --verify-no-changes --verbosity diagnostic
        
    - name: Run code analysis
      run: |
        dotnet tool install --global dotnet-analyzer
        dotnet analyze --verbosity normal
        
    - name: Check database migrations
      run: |
        dotnet tool install --global dotnet-ef
        dotnet ef migrations list --project src/MyApp.Infrastructure

  # Deploy to Staging Job
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build-and-test, security-analysis, code-quality]
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
    - name: Download build artifacts
      uses: actions/download-artifact@v4
      with:
        name: build-artifacts
        
    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v3
      with:
        app-name: 'myapp-staging'
        package: publish/
        azure-credentials: ${{ secrets.AZURE_CREDENTIALS }}
        
    - name: Run smoke tests
      run: |
        # Wait for deployment
        sleep 30
        
        # Health check
        curl -f https://myapp-staging.azurewebsites.net/health
        
        # Basic functionality test
        curl -f https://myapp-staging.azurewebsites.net/api/version
        
    - name: Run integration tests
      run: |
        dotnet test --filter Category=Integration \
          --logger trx \
          --results-directory $(pwd)/IntegrationTestResults
          
    - name: Upload integration test results
      uses: actions/upload-artifact@v4
      with:
        name: integration-test-results
        path: IntegrationTestResults/

  # Deploy to Production Job
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
    - name: Download build artifacts
      uses: actions/download-artifact@v4
      with:
        name: build-artifacts
        
    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v3
      with:
        app-name: 'myapp-production'
        package: publish/
        azure-credentials: ${{ secrets.AZURE_CREDENTIALS }}
        
    - name: Verify deployment
      run: |
        # Wait for deployment
        sleep 30
        
        # Health check
        curl -f https://myapp-production.azurewebsites.net/health
        
        # Performance check
        curl -w "@curl-format.txt" -o /dev/null -s https://myapp-production.azurewebsites.net/api/version
        
    - name: Run post-deployment tests
      run: |
        # Load test
        dotnet tool install --global NBomber
        nbomber run --config nbomber.json
        
    - name: Notify deployment success
      run: |
        # Send notification to Slack/Teams
        curl -X POST -H 'Content-type: application/json' \
          --data '{"text":"Production deployment successful! 🚀"}' \
          ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Testing Integration

### 1. Automated Testing Pipeline
Testing pipeline'ları için configuration.

```csharp
public class TestingPipelineConfiguration
{
    public static void ConfigureTestingServices(IServiceCollection services, IConfiguration configuration)
    {
        // Unit Testing
        services.AddScoped<IUnitTestRunner, UnitTestRunner>();
        
        // Integration Testing
        services.AddScoped<IIntegrationTestRunner, IntegrationTestRunner>();
        
        // Performance Testing
        services.AddScoped<IPerformanceTestRunner, PerformanceTestRunner>();
        
        // Security Testing
        services.AddScoped<ISecurityTestRunner, SecurityTestRunner>();
    }
}

public interface ITestRunner
{
    Task<TestResult> RunTestsAsync(string testType, TestConfiguration config);
}

public class UnitTestRunner : ITestRunner
{
    private readonly ILogger<UnitTestRunner> _logger;
    
    public UnitTestRunner(ILogger<UnitTestRunner> logger)
    {
        _logger = logger;
    }
    
    public async Task<TestResult> RunTestsAsync(string testType, TestConfiguration config)
    {
        try
        {
            _logger.LogInformation("Starting unit tests execution");
            
            var testResult = new TestResult
            {
                TestType = "Unit",
                StartTime = DateTime.UtcNow,
                Status = TestStatus.Running
            };
            
            // Run tests using xUnit or NUnit
            var process = new Process
            {
                StartInfo = new ProcessStartInfo
                {
                    FileName = "dotnet",
                    Arguments = $"test --filter Category=Unit --logger trx --results-directory {config.ResultsDirectory}",
                    WorkingDirectory = config.ProjectPath,
                    RedirectStandardOutput = true,
                    RedirectStandardError = true,
                    UseShellExecute = false
                }
            };
            
            process.Start();
            var output = await process.StandardOutput.ReadToEndAsync();
            var error = await process.StandardError.ReadToEndAsync();
            await process.WaitForExitAsync();
            
            testResult.EndTime = DateTime.UtcNow;
            testResult.Duration = testResult.EndTime - testResult.StartTime;
            testResult.ExitCode = process.ExitCode;
            testResult.Output = output;
            testResult.Error = error;
            testResult.Status = process.ExitCode == 0 ? TestStatus.Passed : TestStatus.Failed;
            
            _logger.LogInformation("Unit tests completed with status: {Status}", testResult.Status);
            return testResult;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error running unit tests");
            return new TestResult
            {
                TestType = "Unit",
                Status = TestStatus.Error,
                Error = ex.Message
            };
        }
    }
}

public class IntegrationTestRunner : ITestRunner
{
    private readonly ILogger<IntegrationTestRunner> _logger;
    private readonly IConfiguration _configuration;
    
    public IntegrationTestRunner(
        ILogger<IntegrationTestRunner> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }
    
    public async Task<TestResult> RunTestsAsync(string testType, TestConfiguration config)
    {
        try
        {
            _logger.LogInformation("Starting integration tests execution");
            
            var testResult = new TestResult
            {
                TestType = "Integration",
                StartTime = DateTime.UtcNow,
                Status = TestStatus.Running
            };
            
            // Setup test environment
            await SetupTestEnvironmentAsync();
            
            // Run integration tests
            var process = new Process
            {
                StartInfo = new ProcessStartInfo
                {
                    FileName = "dotnet",
                    Arguments = $"test --filter Category=Integration --logger trx --results-directory {config.ResultsDirectory}",
                    WorkingDirectory = config.ProjectPath,
                    RedirectStandardOutput = true,
                    RedirectStandardError = true,
                    UseShellExecute = false,
                    EnvironmentVariables = 
                    {
                        ["ASPNETCORE_ENVIRONMENT"] = "Testing",
                        ["ConnectionStrings__DefaultConnection"] = config.TestConnectionString
                    }
                }
            };
            
            process.Start();
            var output = await process.StandardOutput.ReadToEndAsync();
            var error = await process.StandardError.ReadToEndAsync();
            await process.WaitForExitAsync();
            
            // Cleanup test environment
            await CleanupTestEnvironmentAsync();
            
            testResult.EndTime = DateTime.UtcNow;
            testResult.Duration = testResult.EndTime - testResult.StartTime;
            testResult.ExitCode = process.ExitCode;
            testResult.Output = output;
            testResult.Error = error;
            testResult.Status = process.ExitCode == 0 ? TestStatus.Passed : TestStatus.Failed;
            
            _logger.LogInformation("Integration tests completed with status: {Status}", testResult.Status);
            return testResult;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error running integration tests");
            return new TestResult
            {
                TestType = "Integration",
                Status = TestStatus.Error,
                Error = ex.Message
            };
        }
    }
    
    private async Task SetupTestEnvironmentAsync()
    {
        // Setup test database
        // Setup test services
        // Setup test data
        await Task.Delay(1000); // Simulate setup time
    }
    
    private async Task CleanupTestEnvironmentAsync()
    {
        // Cleanup test data
        // Cleanup test services
        await Task.Delay(500); // Simulate cleanup time
    }
}

public class TestResult
{
    public string TestType { get; set; }
    public TestStatus Status { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public TimeSpan Duration { get; set; }
    public int ExitCode { get; set; }
    public string Output { get; set; }
    public string Error { get; set; }
}

public enum TestStatus
{
    NotStarted,
    Running,
    Passed,
    Failed,
    Error
}

public class TestConfiguration
{
    public string ProjectPath { get; set; }
    public string ResultsDirectory { get; set; }
    public string TestConnectionString { get; set; }
    public int TimeoutMinutes { get; set; } = 30;
}
```

### 2. Test Results Processing
Test sonuçlarını işleyen ve raporlayan servis.

```csharp
public class TestResultsProcessor
{
    private readonly ILogger<TestResultsProcessor> _logger;
    private readonly IConfiguration _configuration;
    
    public TestResultsProcessor(
        ILogger<TestResultsProcessor> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }
    
    public async Task<TestSummary> ProcessTestResultsAsync(string resultsDirectory)
    {
        try
        {
            _logger.LogInformation("Processing test results from {Directory}", resultsDirectory);
            
            var testSummary = new TestSummary
            {
                ProcessedAt = DateTime.UtcNow,
                TotalTests = 0,
                PassedTests = 0,
                FailedTests = 0,
                SkippedTests = 0,
                TestResults = new List<TestResultDetail>()
            };
            
            var trxFiles = Directory.GetFiles(resultsDirectory, "*.trx", SearchOption.AllDirectories);
            
            foreach (var trxFile in trxFiles)
            {
                var fileResults = await ProcessTrxFileAsync(trxFile);
                testSummary.TestResults.AddRange(fileResults);
                
                testSummary.TotalTests += fileResults.Count;
                testSummary.PassedTests += fileResults.Count(r => r.Outcome == "Passed");
                testSummary.FailedTests += fileResults.Count(r => r.Outcome == "Failed");
                testSummary.SkippedTests += fileResults.Count(r => r.Outcome == "NotExecuted");
            }
            
            // Calculate success rate
            testSummary.SuccessRate = testSummary.TotalTests > 0 
                ? (double)testSummary.PassedTests / testSummary.TotalTests * 100 
                : 0;
            
            _logger.LogInformation("Test results processed. Total: {Total}, Passed: {Passed}, Failed: {Failed}, Success Rate: {SuccessRate:F2}%",
                testSummary.TotalTests, testSummary.PassedTests, testSummary.FailedTests, testSummary.SuccessRate);
            
            return testSummary;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing test results");
            throw;
        }
    }
    
    private async Task<List<TestResultDetail>> ProcessTrxFileAsync(string trxFile)
    {
        var results = new List<TestResultDetail>();
        
        try
        {
            var xmlDoc = new XmlDocument();
            xmlDoc.Load(trxFile);
            
            var testResults = xmlDoc.SelectNodes("//UnitTestResult");
            
            foreach (XmlNode testResult in testResults)
            {
                var result = new TestResultDetail
                {
                    TestName = testResult.Attributes["testName"]?.Value,
                    Outcome = testResult.Attributes["outcome"]?.Value,
                    Duration = TimeSpan.Parse(testResult.Attributes["duration"]?.Value ?? "00:00:00"),
                    StartTime = DateTime.Parse(testResult.Attributes["startTime"]?.Value ?? DateTime.UtcNow.ToString()),
                    EndTime = DateTime.Parse(testResult.Attributes["endTime"]?.Value ?? DateTime.UtcNow.ToString()),
                    ComputerName = testResult.Attributes["computerName"]?.Value,
                    TestListId = testResult.Attributes["testListId"]?.Value
                };
                
                // Extract error message if test failed
                if (result.Outcome == "Failed")
                {
                    var outputNode = testResult.SelectSingleNode(".//Output");
                    if (outputNode != null)
                    {
                        var errorInfoNode = outputNode.SelectSingleNode(".//ErrorInfo");
                        if (errorInfoNode != null)
                        {
                            var messageNode = errorInfoNode.SelectSingleNode(".//Message");
                            var stackTraceNode = errorInfoNode.SelectSingleNode(".//StackTrace");
                            
                            result.ErrorMessage = messageNode?.InnerText;
                            result.StackTrace = stackTraceNode?.InnerText;
                        }
                    }
                }
                
                results.Add(result);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing TRX file {File}", trxFile);
        }
        
        return results;
    }
    
    public async Task GenerateTestReportAsync(TestSummary summary, string outputPath)
    {
        try
        {
            var report = new StringBuilder();
            
            report.AppendLine("# Test Execution Report");
            report.AppendLine($"Generated: {summary.ProcessedAt:yyyy-MM-dd HH:mm:ss UTC}");
            report.AppendLine();
            
            report.AppendLine("## Summary");
            report.AppendLine($"- Total Tests: {summary.TotalTests}");
            report.AppendLine($"- Passed: {summary.PassedTests}");
            report.AppendLine($"- Failed: {summary.FailedTests}");
            report.AppendLine($"- Skipped: {summary.SkippedTests}");
            report.AppendLine($"- Success Rate: {summary.SuccessRate:F2}%");
            report.AppendLine();
            
            if (summary.FailedTests > 0)
            {
                report.AppendLine("## Failed Tests");
                var failedTests = summary.TestResults.Where(r => r.Outcome == "Failed");
                
                foreach (var test in failedTests)
                {
                    report.AppendLine($"### {test.TestName}");
                    report.AppendLine($"- Duration: {test.Duration}");
                    report.AppendLine($"- Error: {test.ErrorMessage}");
                    if (!string.IsNullOrEmpty(test.StackTrace))
                    {
                        report.AppendLine($"- Stack Trace: ```{test.StackTrace}```");
                    }
                    report.AppendLine();
                }
            }
            
            await File.WriteAllTextAsync(outputPath, report.ToString());
            _logger.LogInformation("Test report generated at {Path}", outputPath);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error generating test report");
            throw;
        }
    }
}

public class TestSummary
{
    public DateTime ProcessedAt { get; set; }
    public int TotalTests { get; set; }
    public int PassedTests { get; set; }
    public int FailedTests { get; set; }
    public int SkippedTests { get; set; }
    public double SuccessRate { get; set; }
    public List<TestResultDetail> TestResults { get; set; } = new List<TestResultDetail>();
}

public class TestResultDetail
{
    public string TestName { get; set; }
    public string Outcome { get; set; }
    public TimeSpan Duration { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public string ComputerName { get; set; }
    public string TestListId { get; set; }
    public string ErrorMessage { get; set; }
    public string StackTrace { get; set; }
}
```

## Deployment Strategies

### 1. Blue-Green Deployment
Blue-green deployment implementasyonu.

```csharp
public class BlueGreenDeploymentService
{
    private readonly ILogger<BlueGreenDeploymentService> _logger;
    private readonly IConfiguration _configuration;
    private readonly IAzureWebAppService _azureWebAppService;
    private readonly ILoadBalancerService _loadBalancerService;
    
    public BlueGreenDeploymentService(
        ILogger<BlueGreenDeploymentService> logger,
        IConfiguration configuration,
        IAzureWebAppService azureWebAppService,
        ILoadBalancerService loadBalancerService)
    {
        _logger = logger;
        _configuration = configuration;
        _azureWebAppService = azureWebAppService;
        _loadBalancerService = loadBalancerService;
    }
    
    public async Task<DeploymentResult> DeployAsync(DeploymentRequest request)
    {
        try
        {
            _logger.LogInformation("Starting blue-green deployment for {Application}", request.ApplicationName);
            
            var deploymentResult = new DeploymentResult
            {
                DeploymentId = Guid.NewGuid().ToString(),
                StartTime = DateTime.UtcNow,
                Status = DeploymentStatus.InProgress
            };
            
            // Determine current active environment
            var currentEnvironment = await DetermineCurrentEnvironmentAsync(request.ApplicationName);
            var targetEnvironment = currentEnvironment == "blue" ? "green" : "blue";
            
            _logger.LogInformation("Current environment: {Current}, Target environment: {Target}", 
                currentEnvironment, targetEnvironment);
            
            // Deploy to target environment
            var deploymentSuccess = await DeployToEnvironmentAsync(request, targetEnvironment);
            if (!deploymentSuccess)
            {
                deploymentResult.Status = DeploymentStatus.Failed;
                deploymentResult.ErrorMessage = "Failed to deploy to target environment";
                return deploymentResult;
            }
            
            // Run health checks on target environment
            var healthCheckPassed = await RunHealthChecksAsync(request.ApplicationName, targetEnvironment);
            if (!healthCheckPassed)
            {
                deploymentResult.Status = DeploymentStatus.Failed;
                deploymentResult.ErrorMessage = "Health checks failed on target environment";
                return deploymentResult;
            }
            
            // Run smoke tests
            var smokeTestsPassed = await RunSmokeTestsAsync(request.ApplicationName, targetEnvironment);
            if (!smokeTestsPassed)
            {
                deploymentResult.Status = DeploymentStatus.Failed;
                deploymentResult.ErrorMessage = "Smoke tests failed on target environment";
                return deploymentResult;
            }
            
            // Switch traffic to target environment
            var trafficSwitchSuccess = await SwitchTrafficAsync(request.ApplicationName, targetEnvironment);
            if (!trafficSwitchSuccess)
            {
                deploymentResult.Status = DeploymentStatus.Failed;
                deploymentResult.ErrorMessage = "Failed to switch traffic to target environment";
                return deploymentResult;
            }
            
            // Monitor target environment
            var monitoringSuccess = await MonitorEnvironmentAsync(request.ApplicationName, targetEnvironment);
            if (!monitoringSuccess)
            {
                _logger.LogWarning("Monitoring detected issues, initiating rollback");
                await RollbackDeploymentAsync(request.ApplicationName, currentEnvironment);
                
                deploymentResult.Status = DeploymentStatus.RolledBack;
                deploymentResult.ErrorMessage = "Deployment rolled back due to monitoring issues";
                return deploymentResult;
            }
            
            // Cleanup old environment (optional)
            if (request.CleanupOldEnvironment)
            {
                await CleanupEnvironmentAsync(request.ApplicationName, currentEnvironment);
            }
            
            deploymentResult.Status = DeploymentStatus.Completed;
            deploymentResult.EndTime = DateTime.UtcNow;
            deploymentResult.Duration = deploymentResult.EndTime - deploymentResult.StartTime;
            deploymentResult.ActiveEnvironment = targetEnvironment;
            
            _logger.LogInformation("Blue-green deployment completed successfully. Active environment: {Environment}", 
                targetEnvironment);
            
            return deploymentResult;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during blue-green deployment");
            return new DeploymentResult
            {
                DeploymentId = Guid.NewGuid().ToString(),
                Status = DeploymentStatus.Failed,
                ErrorMessage = ex.Message
            };
        }
    }
    
    private async Task<string> DetermineCurrentEnvironmentAsync(string applicationName)
    {
        // Check load balancer configuration to determine active environment
        var activeEnvironment = await _loadBalancerService.GetActiveEnvironmentAsync(applicationName);
        return activeEnvironment ?? "blue";
    }
    
    private async Task<bool> DeployToEnvironmentAsync(DeploymentRequest request, string environment)
    {
        try
        {
            var slotName = $"{request.ApplicationName}-{environment}";
            
            // Deploy application to target slot
            var deploymentResult = await _azureWebAppService.DeployAsync(
                request.ApplicationName, 
                slotName, 
                request.PackagePath);
            
            return deploymentResult.Success;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error deploying to environment {Environment}", environment);
            return false;
        }
    }
    
    private async Task<bool> RunHealthChecksAsync(string applicationName, string environment)
    {
        try
        {
            var slotName = $"{applicationName}-{environment}";
            var healthCheckUrl = $"https://{slotName}.azurewebsites.net/health";
            
            // Run health checks multiple times to ensure stability
            for (int i = 0; i < 3; i++)
            {
                using var client = new HttpClient();
                var response = await client.GetAsync(healthCheckUrl);
                
                if (!response.IsSuccessStatusCode)
                {
                    _logger.LogWarning("Health check failed for environment {Environment}, attempt {Attempt}", 
                        environment, i + 1);
                    return false;
                }
                
                await Task.Delay(5000); // Wait 5 seconds between checks
            }
            
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error running health checks for environment {Environment}", environment);
            return false;
        }
    }
    
    private async Task<bool> RunSmokeTestsAsync(string applicationName, string environment)
    {
        try
        {
            var slotName = $"{applicationName}-{environment}";
            var baseUrl = $"https://{slotName}.azurewebsites.net";
            
            // Run basic functionality tests
            var smokeTests = new[]
            {
                $"{baseUrl}/api/version",
                $"{baseUrl}/api/health",
                $"{baseUrl}/api/status"
            };
            
            using var client = new HttpClient();
            
            foreach (var testUrl in smokeTests)
            {
                var response = await client.GetAsync(testUrl);
                if (!response.IsSuccessStatusCode)
                {
                    _logger.LogWarning("Smoke test failed for {Url}", testUrl);
                    return false;
                }
            }
            
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error running smoke tests for environment {Environment}", environment);
            return false;
        }
    }
    
    private async Task<bool> SwitchTrafficAsync(string applicationName, string targetEnvironment)
    {
        try
        {
            // Update load balancer configuration to route traffic to target environment
            var switchResult = await _loadBalancerService.SwitchTrafficAsync(applicationName, targetEnvironment);
            return switchResult.Success;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error switching traffic to environment {Environment}", targetEnvironment);
            return false;
        }
    }
    
    private async Task<bool> MonitorEnvironmentAsync(string applicationName, string environment)
    {
        try
        {
            // Monitor environment for specified duration
            var monitoringDuration = TimeSpan.FromMinutes(5);
            var startTime = DateTime.UtcNow;
            
            while (DateTime.UtcNow - startTime < monitoringDuration)
            {
                var healthCheck = await RunHealthChecksAsync(applicationName, environment);
                if (!healthCheck)
                {
                    return false;
                }
                
                await Task.Delay(30000); // Check every 30 seconds
            }
            
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error monitoring environment {Environment}", environment);
            return false;
        }
    }
    
    private async Task RollbackDeploymentAsync(string applicationName, string rollbackEnvironment)
    {
        try
        {
            _logger.LogInformation("Rolling back deployment to environment {Environment}", rollbackEnvironment);
            
            // Switch traffic back to previous environment
            await _loadBalancerService.SwitchTrafficAsync(applicationName, rollbackEnvironment);
            
            _logger.LogInformation("Rollback completed successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during rollback to environment {Environment}", rollbackEnvironment);
        }
    }
    
    private async Task CleanupEnvironmentAsync(string applicationName, string environment)
    {
        try
        {
            _logger.LogInformation("Cleaning up environment {Environment}", environment);
            
            // Stop the old environment to save costs
            await _azureWebAppService.StopAsync(applicationName, $"{applicationName}-{environment}");
            
            _logger.LogInformation("Environment cleanup completed");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error cleaning up environment {Environment}", environment);
        }
    }
}

public class DeploymentRequest
{
    public string ApplicationName { get; set; }
    public string PackagePath { get; set; }
    public bool CleanupOldEnvironment { get; set; } = true;
    public TimeSpan HealthCheckTimeout { get; set; } = TimeSpan.FromMinutes(5);
    public int MaxRetries { get; set; } = 3;
}

public class DeploymentResult
{
    public string DeploymentId { get; set; }
    public DeploymentStatus Status { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public TimeSpan Duration { get; set; }
    public string ActiveEnvironment { get; set; }
    public string ErrorMessage { get; set; }
}

public enum DeploymentStatus
{
    NotStarted,
    InProgress,
    Completed,
    Failed,
    RolledBack
}
```

## Mülakat Soruları

### Temel Sorular

1. **CI/CD nedir ve neden önemlidir?**
   - **Cevap**: Continuous Integration/Continuous Deployment, code quality, automated testing, reliable deployment, faster delivery.

2. **CI vs CD arasındaki fark nedir?**
   - **Cevap**: CI code integration ve testing, CD deployment automation. CI build ve test, CD deploy ve release.

3. **Pipeline stages nelerdir?**
   - **Cevap**: Build, Test, Security, Quality, Deploy Staging, Deploy Production, Monitoring.

4. **Blue-green deployment nedir?**
   - **Cevap**: Zero-downtime deployment strategy, iki environment, traffic switching, rollback capability.

5. **Pipeline security nasıl sağlanır?**
   - **Cevap**: Secrets management, security scanning, code analysis, access control, audit logging.

### Teknik Sorular

1. **Multi-stage pipeline nasıl tasarlanır?**
   - **Cevap**: Stage dependencies, conditional execution, parallel jobs, artifact sharing, environment management.

2. **Test automation pipeline'da nasıl implement edilir?**
   - **Cevap**: Unit tests, integration tests, smoke tests, performance tests, test result processing.

3. **Deployment rollback nasıl implement edilir?**
   - **Cevap**: Health monitoring, automatic rollback, manual rollback, rollback triggers, state management.

4. **Pipeline monitoring ve alerting nasıl yapılır?**
   - **Cevap**: Pipeline metrics, failure notifications, performance monitoring, deployment tracking.

5. **Pipeline optimization nasıl yapılır?**
   - **Cevap**: Parallel execution, caching, artifact optimization, dependency management, resource utilization.

## Best Practices

1. **Pipeline Design**
   - Multi-stage pipeline kullanın
   - Conditional execution implement edin
   - Parallel jobs optimize edin
   - Artifact sharing yapın

2. **Testing Integration**
   - Automated testing implement edin
   - Test result processing yapın
   - Coverage reporting ekleyin
   - Quality gates kurun

3. **Security & Quality**
   - Security scanning ekleyin
   - Code analysis yapın
   - Secrets management implement edin
   - Access control sağlayın

4. **Deployment Strategy**
   - Blue-green deployment kullanın
   - Health checks implement edin
   - Rollback capability sağlayın
   - Monitoring ekleyin

5. **Monitoring & Alerting**
   - Pipeline metrics toplayın
   - Failure notifications kurun
   - Performance monitoring yapın
   - Deployment tracking implement edin

## Kaynaklar

- [Azure DevOps Documentation](https://docs.microsoft.com/en-us/azure/devops/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [CI/CD Best Practices](https://cloud.google.com/architecture/devops)
- [Pipeline Design Patterns](https://www.jenkins.io/doc/book/pipeline/)
- [Deployment Strategies](https://martinfowler.com/bliki/BlueGreenDeployment.html)
