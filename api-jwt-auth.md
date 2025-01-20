# JWT Auth for API endpoints
Add a simple API integration to your serenity .NET core project, using JWT authentication

## Create an endpoint that generates a token, in ApiAccountController.cs

```
namespace MyProject.Membership.Pages
{
    ...
    using Microsoft.AspNetCore.Authentication.JwtBearer;
 
    [Authorize(AuthenticationSchemes = JwtBearerDefaults.AuthenticationScheme)]
    [Route("Api/Account/[action]")]
    public partial class AccountApiController : Controller
    {
        [AllowAnonymous]
        [HttpPost]
        [IgnoreAntiforgeryToken]
        public async Task<IActionResult> GenerateToken(
            [FromBody] LoginRequest request,
            [FromServices] IUserPasswordValidator passwordValidator,
            [FromServices] IUserRetrieveService userRetriever)
        {
            if (request is null)
                throw new ArgumentNullException(nameof(request));

            if (string.IsNullOrEmpty(request.Username))
                throw new ArgumentNullException("username");

            if (passwordValidator is null)
                throw new ArgumentNullException(nameof(passwordValidator));

            if (userRetriever is null)
                throw new ArgumentNullException(nameof(userRetriever));

            var username = request.Username;
            var passwordValidationResult = PasswordValidationResult.Valid;
            await Task.Run(() =>
            {
                passwordValidationResult = passwordValidator.Validate(ref username, request.Password);
            });

            if (passwordValidationResult != PasswordValidationResult.Valid && passwordValidationResult != PasswordValidationResult.PasswordChangeRequired)
                return BadRequest("Could not create token");

            var principal = UserRetrieveService.CreatePrincipal(userRetriever, username, authType: "Password");

          var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("secret key"));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
            var myDomain = Request.Host.Value;

            var token = new JwtSecurityToken(myDomain, myDomain,
              principal.Claims,
              expires: DateTime.Now.AddDays(365),
              signingCredentials: creds);

            return Ok(new
            {
                token = new JwtSecurityTokenHandler().WriteToken(token),
           });
        }
        
        ...
```
## Configure startup services to accept JWT

```
 ...
     services.AddAuthentication(o =>
            {
            ....
            })
            .AddJwtBearer(options =>
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = false, //to by-pass issuer validation
                    ValidateAudience = false,
                    ValidateIssuerSigningKey = true,
                    //ValidIssuer = TokenHelper.Issuer, // beyond scope of this article
                    //ValidAudience = TokenHelper.Audience,
                    IssuerSigningKey = new SymmetricSecurityKey(Convert.FromBase64String(TokenHelper.Secret))
                };

            });
```            
## Create endpoint API unit 

```
namespace myproject.Api
{
    [IgnoreAntiforgeryToken]
    [ApiController]
    [Route("api/")]
    [Authorize(AuthenticationSchemes = JwtBearerDefaults.AuthenticationScheme)]
    public class MyApi : ControllerBase
    {
        [HttpGet, Route("myendpoint")]
        public IActionResult MyEndpoint([FromQuery] int? value)
        {
        ...
         return Ok($"value is {value}");
         }
     }
```