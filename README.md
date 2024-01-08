# Text-File-Repo
For Coding Data
import net.bytebuddy.pool.TypePool;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.support.ui.Select;
import org.openqa.selenium.support.ui.Wait;
import org.openqa.selenium.support.ui.WebDriverWait;
import java.time.Duration;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.TimeUnit;

public class Blend_Stage_Add_Account {
    public static void main(String[] args) throws InterruptedException {

        System.setProperty("webdriver.chrome.driver", "C:\\KRISHNAN WORK DATA\\KRISHNAN\\Automation_Repository\\Drivers\\chromedriver-win64\\chromedriver.exe");
        WebDriver driver = new ChromeDriver();
        //driver.get("https://wmstg-wm.onefiserv.net/Stub/widgets/jsp/angularStub.jsp");
        // To launch Prod use main QA Domain
        driver.get("https://aggmnqa-wm.onefiserv.net/Stub/widgets/jsp/angularStub.jsp");
        driver.manage().timeouts().implicitlyWait(60, TimeUnit.SECONDS);
        driver.manage().window().maximize();
// Dropdown to choose stage
        WebElement drop = driver.findElement(By.id("environment"));
        Select dropdown = new Select(drop);
        dropdown.selectByVisibleText("StageQA");
// session token
        driver.findElement(By.name("sessionToken")).sendKeys("61732b306b5173595141494b74562f466352357355413d3d");
        driver.findElement(By.name("error_url")).sendKeys("https://aggmnqa-wm.onefiserv.net/PFM_UI/login/88886648");
        driver.findElement(By.name("return_url")).sendKeys("https://aggmnqa-wm.onefiserv.net/PFM_UI/login/88886648");
       /* driver.findElement(By.name("css_url")).sendKeys("https://aggmnqa-wm.onefiserv.net/PFM_UI/login/88886648");
        driver.findElement(By.name("keepalive_url")).sendKeys("https://aggmnqa-wm.onefiserv.net/PFM_UI/login/88886648");*/
        driver.findElement(By.name("partner_app_id")).sendKeys("Blend1");
        String parentWindowHandle = driver.getWindowHandle();
        driver.findElement(By.xpath("//input[@type='submit']")).click();
        // Calling Child window
        String childwindow = getChildWindowHandle(driver, parentWindowHandle);
        if (childwindow != null)
        {
            driver.switchTo().window(childwindow);
        }
        /*// Navigating to the newly opened child Window.......
        Set<String> windows = driver.getWindowHandles();
        System.out.println(windows);
        Iterator it = windows.iterator();
        String parentwindow = (String) it.next();
        String childwindow = (String) it.next();
        driver.switchTo().window(childwindow);*/
        // Fiserv Pfm Widgets Add account flow .
        driver.findElement(By.id("searchbar")).sendKeys("pfm widget");
        driver.findElement(By.id("ul-div-id-1")).click();
        driver.findElement(By.id("userFiLogin")).sendKeys("Krish_widget1");
        driver.findElement(By.id("userFiPassword")).sendKeys("Krish_widget1");
        driver.findElement(By.xpath("//button[@class='btn btn-primary']")).click();
        Thread.sleep(Duration.ofSeconds(50));
        String PageTitle = driver.getTitle();
        System.out.println(PageTitle);
        String ReturnURLwithchildparentids = driver.getCurrentUrl();
        System.out.println(ReturnURLwithchildparentids);
        driver.close();
        // Switching to parent window and launching the add account widget again
        driver.switchTo().window(parentWindowHandle);
        driver.findElement(By.xpath("//input[@type='submit']")).click();
        /*// Navigating to the newly opened child Window.......
        Set<String> windows1 = driver.getWindowHandles();
        System.out.println(windows1);
        Iterator itr = windows1.iterator();
        String parentwindow1 = (String) itr.next();
        String childwindow1 = (String) itr.next();
        driver.switchTo().window(childwindow1);*/
        // Calling Child window
        childwindow = getChildWindowHandle(driver, parentWindowHandle);
        if (childwindow != null) {
            driver.switchTo().window(childwindow);
        }
        // Cashedge Bank Retail Mfa FI  Add account flow .
        driver.findElement(By.id("searchbar")).sendKeys("Retail mfa");
        driver.findElement(By.id("ul-div-id-2")).click();
        driver.findElement(By.id("userFiLogin")).sendKeys("MFA_User01");
        driver.findElement(By.id("userFiPassword")).sendKeys("MFA_User01");
        Wait<WebDriver> wait = new WebDriverWait(driver, Duration.ofSeconds(60));
        driver.findElement(By.xpath("//button[@type='submit']")).click();
        WebElement mfabox = driver.findElement(By.xpath("//input[@type='password']"));
        wait.until(d -> mfabox.isDisplayed());
        mfabox.sendKeys("baseball");
        driver.findElement(By.xpath("//button[@class='btn btn-primary']")).click();
        Thread.sleep(Duration.ofSeconds(50));
        PageTitle = driver.getTitle();
        System.out.println(PageTitle);
        ReturnURLwithchildparentids = driver.getCurrentUrl();
        System.out.println(ReturnURLwithchildparentids);
        driver.close();
        // Switching to parent window and launching the add account widget again
        driver.switchTo().window(parentWindowHandle);
        driver.findElement(By.xpath("//input[@type='submit']")).click();
        // Calling Child window
        childwindow = getChildWindowHandle(driver, parentWindowHandle);
        if (childwindow != null) {
            driver.switchTo().window(childwindow);
        }
        // Cashedge Bank Retail Non Mfa FI  Add account flow .
        driver.findElement(By.id("searchbar")).sendKeys("non mfa");
        driver.findElement(By.id("ul-div-id-1")).click();
        driver.findElement(By.id("userFiLogin")).sendKeys("New_cred");
        driver.findElement(By.id("userFiPassword")).sendKeys("New_cred");
        Thread.sleep(Duration.ofSeconds(10));
        driver.manage().timeouts().implicitlyWait(60, TimeUnit.SECONDS);
        driver.findElement(By.xpath("//button[@class='btn btn-primary']")).click();
        Thread.sleep(Duration.ofSeconds(30));
        PageTitle = driver.getTitle();
        System.out.println(PageTitle);
        ReturnURLwithchildparentids = driver.getCurrentUrl();
        System.out.println(ReturnURLwithchildparentids);
        driver.close();
        driver.quit();
    }
    public static String getChildWindowHandle(WebDriver driver, String parentWindowHandle) {
        Set<String> handles = driver.getWindowHandles();
        for (Iterator itr = handles.iterator(); itr.hasNext(); )
        {
            String currentHandle = (String) itr.next();
            if (!parentWindowHandle.equals(currentHandle))
                return currentHandle;
        }
        return null;
    }
}
