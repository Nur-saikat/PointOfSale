
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.Data.SqlClient;
using Serenity;
using Serenity.Abstractions;
using Serenity.Data;
using Serenity.Extensions;
using Serenity.Services;
using SmartERP.Administration;
using SmartERP.Administration.Entities;
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using System.Security.Claims;

namespace SmartERP.Membership.Pages
{
    [Route("Account/[action]")]
    public partial class AccountController : Controller
    {
        public ITwoLevelCache Cache { get; }
        protected ISqlConnections SqlConnections { get; }
        public ITextLocalizer Localizer { get; }

        public AccountController(ITwoLevelCache cache, ITextLocalizer localizer, ISqlConnections sqlConnections)
        {
            Cache = cache ?? throw new ArgumentNullException(nameof(cache));
            Localizer = localizer ?? throw new ArgumentNullException(nameof(localizer));
            SqlConnections = sqlConnections;
        }
        public static bool UseAdminLTELoginBox = false;

        [HttpGet]
        public ActionResult Login(string activated)
        {
            ViewData["Activated"] = activated;
            ViewData["HideLeftNavigation"] = true;

            if (UseAdminLTELoginBox)
                return View(MVC.Views.Membership.Account.AccountLogin_AdminLTE);
            else
                return View(MVC.Views.Membership.Account.AccountLogin);
        }

        [HttpGet]
        public ActionResult AccessDenied(string returnURL)
        {
            ViewData["HideLeftNavigation"] = !User.IsLoggedIn();

            return View(MVC.Views.Errors.AccessDenied, (object)returnURL);
        }

        [HttpPost, JsonRequest]
        public Result<ServiceResponse> Login(LoginRequest request,
            [FromServices] IUserPasswordValidator passwordValidator,
            [FromServices] IUserRetrieveService userRetriever,
            [FromServices] IEmailSender emailSender = null)
        {
            return this.ExecuteMethod(() =>
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
                var result = passwordValidator.Validate(ref username, request.Password);
                if (result == PasswordValidationResult.Valid)
                {
                    using var connection = SqlConnections.NewFor<UserRoleRow>();
                    var user = userRetriever.ByUsername(username);
                    if (user == null)
                        throw new ValidationError("AuthenticationError", Texts.Validation.AuthenticationError.ToString(Localizer));

                    var companyIds = connection.List<UserRoleRow>(
                        new Criteria(UserRoleRow.Fields.UserId) == user.Id)
                    .Select(x => x.CompanieId)
                    .Where(id => id != null)
                    .Distinct()
                    .ToList();

                    if (!companyIds.Any())
                    {
                        var principal = UserRetrieveService.CreatePrincipal(userRetriever, username, authType: "Password");
                        HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, principal)
                            .GetAwaiter().GetResult();
                        return new ServiceResponse();
                    }

                    if (companyIds.Count == 1)
                    {
                        var singleCompanyId = companyIds.First().Value; // Because it's int?

                        var principal = UserRetrieveService.CreatePrincipal(userRetriever, username, authType: "Password");
                        ((ClaimsIdentity)principal.Identity).AddClaim(new Claim("CompanieId", singleCompanyId.ToString()));

                        HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, principal)
                            .GetAwaiter().GetResult();

                        return new ServiceResponse();
                    }

                    // If multiple companies, return them to client
                    var companies = connection.Query<CompaniesRow>(@"SELECT DISTINCT c.Id, c.CompanyName FROM UserRoles ur JOIN Companies c ON c.Id = ur.CompanieId WHERE ur.UserId = @UserId", new { UserId = user.Id }).ToList();

                    return new ServiceResponse
                    {
                        CustomData = new Dictionary<string, object>
                        {
                            ["Companies"] = companies,
                            ["Username"] = username
                        }
                    };


                }

                throw new ValidationError("AuthenticationError", Texts.Validation.AuthenticationError.ToString(Localizer));
            });
        }
       

        private ActionResult Error(string message)
        {
            return View(MVC.Views.Errors.ValidationError, message);
        }

        public ActionResult Signout()
        {
            HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
            return new RedirectResult("~/");
        }
    }
}
