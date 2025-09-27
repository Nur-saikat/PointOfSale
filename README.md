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
