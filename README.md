Edi.TemplateEmail
===============================

This library enable you to configure email in XML template and send the email in your .NET / .NET Core applications.

## For .NET Core

Notice: .NET Core support is done via .NET Standard 2.0, it can support .NET Framework, but not recommended for this package yet.

```
PM > Install-Package Edi.TemplatedEmail.NetStd
```

### Step 1: Sample mailConfiguration.xml under your application root

```xml
<?xml version="1.0"?>
<MailConfiguration>
  <CommonConfiguration OverrideToAddress="false" ToAddress="overridetest@test.com" />
  <MailMessage MessageType="TestMail" IsHtml="true">
    <MessageSubject>Test Mail on {MachineName.Value}</MessageSubject>
    <MessageBody>
      <![CDATA[
Mail configuration on {MachineName.Value} is good: <br />
Smtp Server: {SmtpServer.Value}<br />
Smtp Port: {SmtpServerPort.Value}<br />
Smtp Username: {SmtpUserName.Value}<br />
Email Display Name: {EmailDisplayName.Value}<br />
Enable SSL: {EnableSsl.Value}<br />
      ]]>
    </MessageBody>
  </MailMessage>
</MailConfiguration>
```

Optional: You may need to set it to copy to output directory based on your usage.

### Step 2:

Define an EmailHelper object in your class.
```
public static EmailHelper EmailHelper { get; set; }
```

Get the configuration file path

```
var configSource = $"{Directory.GetCurrentDirectory()}\\mailConfiguration.config";
```

Initialize the EmailHelper by your mail server settings

```
var settings = new EmailSettings("smtp-mail.outlook.com", "Edi.Test@outlook.com", "YOURPASSWORD", 587)
{
    EnableSsl = true,
    EmailDisplayName = "Edi.TemplateEmail.NetStd",
    SenderName = "Test Sender"
};

EmailHelper = new EmailHelper(configSource, settings);
```

Optional: You can also define event handlers on this

```
EmailHelper.EmailSent += (sender, eventArgs) =>
{
    if (sender is MailMessage msg)
        Console.WriteLine($"Email {msg.Subject} is sent, Success: {eventArgs.IsSuccess}");
};
EmailHelper.EmailFailed += (sender, eventArgs) =>
{
    Console.WriteLine("Failed");
};
EmailHelper.EmailCompleted += (sender, e) =>
{
    Console.WriteLine("Completed.");
};
```

### Step 3: Map the Configuration Values and Send Email

```
public static async Task TestSendTestMail()
{
    bool isOk = true;

    var pipeline = new TemplatePipeline().Map("MachineName", Environment.MachineName)
        .Map("SmtpServer", EmailHelper.Settings.SmtpServer)
        .Map("SmtpServerPort", EmailHelper.Settings.SmtpServerPort)
        .Map("SmtpUserName", EmailHelper.Settings.SmtpUserName)
        .Map("EmailDisplayName", EmailHelper.Settings.EmailDisplayName)
        .Map("EnableSsl", EmailHelper.Settings.EnableSsl);

    EmailHelper.EmailFailed += (s, e) =>
    {
        isOk = false;
    };

    await EmailHelper.ApplyTemplate("TestMail", pipeline).SendMailAsync("Edi.Wang@outlook.com");
}
```

## For Legacy .NET Framework

```
PM > Install-Package Edi.TemplatedEmail
```

### Step 1: Sample mailConfiguration.config under your website root

```xml
<?xml version="1.0"?>
<mailConfiguration>
  <CommonConfiguration OverrideToAddress="false" ToAddress="" />
  <MailMessage MessageType="TestMail" IsHtml="true">
    <MessageSubject>Test Mail on {MachineName.Value}</MessageSubject>
    <MessageBody>
      <![CDATA[
Email Sending has been configured on {MachineName.Value} <br />
Smtp Server: {SmtpServer.Value}<br />
Smtp Port: {SmtpServerPort.Value}<br />
Smtp Username: {SmtpUserName.Value}<br />
Display Name: {EmailDisplayName.Value}<br />
Require SSL: {EnableSsl.Value}<br />
      ]]>
    </MessageBody>
  </MailMessage>
  ...
</mailConfiguration>
```

### Step 2: Add Following Section in Web.config

```xml
<configSections>
    ...
    <section name="mailConfiguration" type="Edi.XmlConfigMapper.XmlSection`1[[Edi.TemplateEmail.MailConfiguration, Edi.TemplateEmail, Culture=neutral]], Edi.XmlConfigMapper" />
</configSections>
<mailConfiguration configSource="mailConfiguration.config" />
```

### Step 3: C# Sample Code

```
public static EmailHelper EmailHelper { get; private set; }

static EmailNotification()
{
    if (EmailHelper == null)
    {
        EmailHelper = new EmailHelper(new EmailSettings
        {
            SmtpServer = Settings.Settings.Instance.SmtpServer,
            SmtpUserName = Settings.Settings.Instance.SmtpUserName,
            SmtpPassword = Utils.DecryptEmailPassword(Settings.Settings.Instance.SmtpPassword),
            SmtpServerPort = Settings.Settings.Instance.SmtpServerPort,
            EnableSsl = Settings.Settings.Instance.EnableSslForEmail,
            EmailDisplayName = Settings.Settings.Instance.EmailDisplayName
        })
        .SendAs(PubConstant.SENDERNAME)
        .LogExceptionWith(Logger.Error);

        if (Settings.Settings.Instance.IncludeSignature)
        {
            EmailHelper.UseSignature(Settings.Settings.Instance.SignatureContent);
        }

        if (Settings.Settings.Instance.EmailWithSystemInfo)
        {
            EmailHelper.Settings.EmailWithSystemInfo = true;
            EmailHelper.WithFooter(string.Format("<p>Powered by EdiBlog {0}</p>", Settings.Settings.Instance.Version()));
        }

        EmailHelper.EmailCompleted += (sender, message, args) => WriteEmailLog(sender as MailMessage, message);
    }
}

...

public static void SendTestMail()
{
    var pipeline = new TemplatePipeline().Map("MachineName", HttpContext.Current.Server.MachineName)
                                         .Map("SmtpServer", Settings.Settings.Instance.SmtpServer)
                                         .Map("SmtpServerPort", Settings.Settings.Instance.SmtpServerPort)
                                         .Map("SmtpUserName", Settings.Settings.Instance.SmtpUserName)
                                         .Map("EmailDisplayName", Settings.Settings.Instance.SmtpUserName)
                                         .Map("EnableSsl", Settings.Settings.Instance.EnableSslForEmail);

    Task.Run(() => EmailHelper.ApplyTemplate(MailMesageType.TestMail, pipeline)
        .SendMailAsync(Settings.Settings.Instance.AdminEmail));
}

public static void SendNewCommentNotificationAsync(Comment comment, Post post)
{
    var pipeline = new TemplatePipeline().Map("Username", comment.Username)
                                         .Map("Email", comment.Email)
                                         .Map("IPAddress", comment.IPAddress)
                                         .Map("PubDate", comment.PubDate)
                                         .Map("Title", post.Title)
                                         .Map("CommentContent", comment.CommentContent);
    Task.Run(() => EmailHelper.ApplyTemplate(MailMesageType.NewCommentNotification, pipeline)
        .SendMailAsync(Settings.Settings.Instance.AdminEmail));
}
```

**and you can also**

```
// AfterComplete() will update complete status of the mail for Messages table in db
await EmailHelper.ApplyTemplate(MailMesageType.ContactMessageForAdmin, pipelineForAdmin)
                 .AfterComplete(async () => await new MessageOperator().UpdateCompleteStatusForMessage(mailId))
                 .SendMailAsync(Settings.Settings.Instance.AdminEmail);

// clear after complete in case of other mail methods using this action
EmailHelper.AfterCompleteAction = null;
```