// WebApp, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null
// WebApp.Controllers.CMDBController
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Net;
using System.Text;
using System.Threading.Tasks;
using App.Core.Collection;
using App.Core.Collection.User;
using App.Core.Enum;
using App.Extensions;
using App.Extensions.Helpers;
using App.Extensions.Localization;
using App.Extensions.ResultOptions;
using App.Extensions.Security;
using App.Infrastructure.DbContext;
using App.Service.ConfigFile;
using App.Service.Customer;
using App.Service.Device;
using App.Service.Device.Dto;
using App.Service.EventLog;
using App.Service.InternalData;
using App.Service.IPList;
using App.Service.IPList.Dto;
using App.Service.IPService;
using App.Service.IPService.Dto;
using App.Service.Post;
using App.Service.Service;
using App.Service.Site;
using App.Service.Users;
using DNTBreadCrumb.Core;
using JqGrid.Core.DynamicSearch;
using Lib.AspNetCore.Mvc.JqGrid.Core.Json;
using Lib.AspNetCore.Mvc.JqGrid.Core.Request;
using Lib.AspNetCore.Mvc.JqGrid.Core.Response;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Localization;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using MongoDB.Bson;
using MongoDB.Driver;
using Newtonsoft.Json;
using OfficeOpenXml;
using WebApp;
using WebApp.Controllers;
using WebApp.Helper.Options;
using WebApp.Helper.Services;
using WebApp.Models.AgGrid;
using WebApp.Models.ViewModel;

[Authorize]
[BreadCrumb(Title = "CMDB", UseDefaultRouteUrl = true, Order = 1, GlyphIcon = "icon icon-group-circle")]
[AutoValidateAntiforgeryToken]
[MiddlewareFilter(typeof(LocalizationPipeline))]
public class CMDBController : BaseController
{
	private IInternalDataService _internalDataService;

	private IIPListService _ipListService;

	private IIPServiceService _ipServiceService;

	private ISiteService _siteService;

	private ICustomersService _customerService;

	private IConfigFileService _configFileService;

	private IQueryable<Devices> _myDevices;

	private Dictionary<string, int> ColumnList;

	private IQueryable<IPLists> _myIPAM;

	private IQueryable<IPServices> _myIPServices;

	private List<string> _ipTags;

	private IQueryable<Devices> MyDevices
	{
		get
		{
			if (_myDevices == null)
			{
				bool isAll = base._accessUser.networks.Any((AccessNetwork a) => a.network == "All");
				_myDevices = _cachProvider._AllDevices.Where((Devices tt) => (isAll && tt.tag.network.ToLower() != "unknown") || base._accessUser.networks.Any((AccessNetwork a) => a.network == tt.tag.network)).AsQueryable();
				return _myDevices;
			}
			return _myDevices;
		}
	}

	private IQueryable<IPLists> MyIPAM
	{
		get
		{
			if (_myIPAM == null)
			{
				_myIPAM = _cachProvider._AllIPList.Where((IPLists tt) => !tt.isSecret || base._accessUser.ipams.Any((AccessNetwork a) => a.network == tt.networkName)).AsQueryable();
				return _myIPAM;
			}
			return _myIPAM;
		}
	}

	private IQueryable<IPServices> MyIPServices
	{
		get
		{
			if (_myIPServices == null)
			{
				_myIPServices = _cachProvider._AllIPServices.Where((IPServices tt) => base._accessUser.ipams.Any((AccessNetwork a) => a.network == tt.network)).AsQueryable();
				return _myIPServices;
			}
			return _myIPServices;
		}
	}

	private List<string> ipTags
	{
		get
		{
			_ipTags = new List<string>();
			List<string> list = (from tt in MyIPAM.DistinctBy((IPLists tt) => tt.tags)
				select tt.tags).ToList();
			foreach (string item in list)
			{
				string[] array = item.Split(",");
				foreach (string text in array)
				{
					if (!string.IsNullOrEmpty(text) && !_ipTags.Contains(text))
					{
						_ipTags.Add(text);
					}
				}
			}
			return _ipTags;
		}
	}

	public CMDBController(IEmailSender emailSender, ILogger<CMDBController> logger, IStringLocalizer<Common> localizer, ILanguageResourcesService languageResourcesService, IWebHostEnvironment hostEnvironment, IUserService userService, IMongoDbContext mongoDbContext, IPostService postService, IServiceService serviceService, IIPServiceService ipServiceService, ISiteService siteService, ICustomersService customerService, IInternalDataService internalDataService, IIPListService ipListService, IEventLogsService eventLogsService, IDeviceService deviceService, IConfigFileService configFileService, ICachProvider cachProvider, IOptionsSnapshot<ApplicationSettings> appSettings)
	{
		_mongoDbContext = mongoDbContext;
		_emailSender = emailSender;
		_logger = logger;
		_webHostEnvironment = hostEnvironment;
		_localizer = localizer;
		_postService = postService;
		_userService = userService;
		_eventLogsService = eventLogsService;
		_internalDataService = internalDataService;
		_ipListService = ipListService;
		_ipServiceService = ipServiceService;
		_deviceService = deviceService;
		_serviceService = serviceService;
		_siteService = siteService;
		_customerService = customerService;
		_configFileService = configFileService;
		_appSettings = appSettings;
		_cachProvider = cachProvider;
		_languageResourcesService = languageResourcesService;
	}

	[BreadCrumb(Title = "CMDB", Order = 1, GlyphIcon = "icon icon-group-circle")]
	[AddHeader("CMDB")]
	public ActionResult Index()
	{
		return View();
	}

	[HttpGet]
	[Authorize(Roles = "CMDB")]
	[AddHeader("Services")]
	[BreadCrumb(Title = "Services", Order = 2, GlyphIcon = "icon icon-swap-horizontal ")]
	public IActionResult Services()
	{
		return View();
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<IActionResult> GetServices()
	{
		try
		{
			await AddEventLogAsync("CMDB-Show-Services", base.CurrentUser.mainUserInfo.UserName);
			return new JsonResult(_cachProvider._AllServices.Select((Services service) => new Services
			{
				id = service.id,
				code = service.code,
				nameFa = service.nameFa,
				nameEn = service.nameEn,
				view_networks = service.view_networks,
				view_customer_count = service.view_customer_count,
				info = service.info
			}).ToList());
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetServices():257");
		}
		return new JsonResult(null);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public IActionResult GetCustomersByService([FromForm] GridParam gridParam)
	{
		try
		{
			Services serviceByServiceID = GetServiceByServiceID(ObjectId.Parse(gridParam.id));
			List<ObjectId> customerIDs = serviceByServiceID.customers.Select((IDName s) => s.id).ToList();
			IEnumerable<Customers> enumerable = from tt in _cachProvider._AllCustomers
				where customerIDs.Contains(tt.id)
				select tt into t
				select (t);
			if (enumerable != null)
			{
				return new JsonResult(enumerable.Select((Customers customer) => new Customers
				{
					id = customer.id,
					code = customer.code,
					nameFa = customer.nameFa,
					nameEn = customer.nameEn,
					info = customer.info
				}));
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetCustomersByService():283");
		}
		return new JsonResult(null);
	}

	[HttpGet]
	[Authorize(Roles = "CMDB")]
	[AddHeader("Customers")]
	[BreadCrumb(Title = "Customers", Order = 2, GlyphIcon = "icon icon-clipboard-user ")]
	public IActionResult Customers()
	{
		return View();
	}

	[HttpPost]
	[AjaxOnly]
	[Authorize(Roles = "CMDB")]
	public async Task<JsonResult> GetCustomers()
	{
		try
		{
			await AddEventLogAsync("CMDB-Show-Customers", base.CurrentUser.mainUserInfo.UserName);
			return new JsonResult(_cachProvider._AllCustomers.Select((Customers customer) => new Customers
			{
				id = customer.id,
				code = customer.code,
				nameFa = customer.nameFa,
				nameEn = customer.nameEn,
				view_service_count = customer.view_service_count,
				changeCount = customer.changeCount,
				info = customer.info
			}).ToList());
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetCustomers():522");
		}
		return new JsonResult(null);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public IActionResult GetServicesByCustomer([FromForm] GridParam gridParam)
	{
		try
		{
			List<Services> servicesByCustomerID = GetServicesByCustomerID(ObjectId.Parse(gridParam.id));
			if (servicesByCustomerID != null)
			{
				return new JsonResult(servicesByCustomerID.Select((Services service) => new Services
				{
					id = service.id,
					code = service.code,
					nameFa = service.nameFa,
					nameEn = service.nameEn,
					view_networks = service.view_networks,
					view_customer_count = service.view_customer_count,
					info = service.info
				}));
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetServicesByCustomer():548");
		}
		return new JsonResult(null);
	}

	[HttpGet]
	[AddHeader("Devices")]
	[Authorize(Roles = "CMDB")]
	[BreadCrumb(Title = "Devices", Order = 2, GlyphIcon = "icon icon-database ")]
	public async Task<IActionResult> Devices()
	{
		bool flag = base._accessUser.networks.Any((AccessNetwork a) => a.network == "All");
		base.ViewData["groupNetwork"] = flag || base._accessUser.networks.Count > 2;
		await SetDevicePatchParam();
		List<InternalDatas> source = await _internalDataService.GetAllAsync();
		base.ViewData["RiskFactorItems"] = source.Where((InternalDatas tt) => tt.Category == "riskFactor").ToList();
		base.ViewData["DescriptionTypeItems"] = source.Where((InternalDatas tt) => tt.Category == "descriptionType").ToList();
		base.ViewBag.DeviceServices = _cachProvider._AllIPServices.Select((IPServices tt) => tt.nameEn).ToList();
		return View();
	}

	[HttpGet]
	[AddHeader("Patchs")]
	[Authorize(Roles = "CMDB")]
	[BreadCrumb(Title = "Patchs", Order = 2, GlyphIcon = "icon icon-shield ")]
	public IActionResult Patchs()
	{
		return View();
	}

	[HttpGet]
	[Authorize(Roles = "CMDB")]
	[AddHeader("DownloadConfig")]
	public async Task<FileStreamResult> DownloadConfig(ConfigDownload configParam)
	{
		string fileType = "text/plain";
		string fileName = string.Empty;
		string pathForFile = string.Empty;
		bool hasError = false;
		Devices device = MyDevices.Where((Devices tt) => tt.id == ObjectId.Parse(configParam.devId)).SingleOrDefault();
		ConfigFiles configFiles = await _configFileService.GetAsync(configParam.id);
		if (configFiles != null)
		{
			pathForFile = configFiles.address;
			int num = configFiles.address.LastIndexOf("\\");
			fileName = configFiles.address.Substring(num + 1, configFiles.address.Length - num - 1);
		}
		if (pathForFile == string.Empty || !System.IO.File.Exists(pathForFile) || device == null)
		{
			pathForFile = GetServeUploadPath() + "notAccess.txt";
			fileName = "عدم_دسترسی.txt";
			if (!System.IO.File.Exists(pathForFile))
			{
				fileName = "فایل_وجود_ندارد.txt";
			}
			hasError = true;
		}
		byte[] bytesInStream = ((!hasError) ? (await base._EncryptionManager.DecryptMemoryAsync(pathForFile, base._EncryptionManager.EncryptionKey)).ToArray() : Encoding.UTF8.GetBytes(fileName));
		await AddEventLogAsync("Devices-Config-Download", fileName);
		base.Response.ContentType = fileType;
		base.Response.Headers.Append("content-disposition", "attachment; filename*=UTF-8''" + Uri.EscapeDataString(fileName) + ".txt");
		return File(new MemoryStream(bytesInStream), fileType, fileName + ".txt");
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<JsonResult> GetDevices()
	{
		try
		{
			await AddEventLogAsync("CMDB-Show-Devices", base.CurrentUser.mainUserInfo.UserName);
			return new JsonResult(MyDevices.Select((Devices tt) => new Devices
			{
				id = tt.id,
				name = tt.name,
				type = tt.type,
				gridview_network = tt.tag.network,
				view_tags = tt.view_tags,
				ipManagement = tt.ipManagement,
				series = tt.series,
				model = tt.model,
				view_devint_count = tt.view_devint_count,
				view_patch_count = tt.view_patch_count,
				gridview_patch_last_version = ((tt.view_patch_last != null) ? tt.view_patch_last.requireVersion : string.Empty),
				osVersion = tt.osVersion,
				site = tt.site,
				subSite = tt.subSite,
				view_locations = tt.view_locations,
				gridview_networkLevel = tt.tag.level,
				view_services = tt.view_services,
				view_ACSServerIPs = tt.view_ACSServerIPs,
				label = tt.label,
				propertyNo = tt.propertyNo,
				gridview_logSourceIP = tt.log.sourceIP,
				view_logServers = tt.view_logServers,
				gridview_logSeverity = tt.log.severity,
				gridview_ntpConfigSyncType = tt.ntpConfig.syncType,
				view_ntpConfigIPs = tt.view_ntpConfigIPs,
				backupDevice = tt.backupDevice,
				powerBackup = tt.powerBackup,
				isVirtual = tt.isVirtual,
				localUsername = tt.localUsername,
				ownerGroup = tt.ownerGroup,
				gridview_lastUpdateDate = tt.lastUpdate.date.ToString("yyyy-MM-dd"),
				configFile = tt.configFile
			}).ToList());
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetDevices():743");
		}
		return new JsonResult(null);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public IActionResult GetLinksByDevice([FromForm] GridParam gridParam)
	{
		try
		{
			List<Devint> interfacesByDeviceID = GetInterfacesByDeviceID(ObjectId.Parse(gridParam.id));
			List<ObjectId> devintIDs = new List<ObjectId>(interfacesByDeviceID.Select((Devint s) => s.id).ToList());
			IMongoCollection<Links> collection = _mongoDbContext.GetCollection<Links>("Links");
			List<Links> allLinks = (from tt in collection.AsQueryable()
				where devintIDs.Contains(tt.side1) || devintIDs.Contains(tt.side2)
				select tt).ToList();
			List<DeviceLink> list = new List<DeviceLink>();
			foreach (Devint item in interfacesByDeviceID)
			{
				list.AddRange(GetLinkInterface(item, allLinks));
			}
			if (list != null)
			{
				return new JsonResult(list.Select((DeviceLink tt) => new DeviceLink
				{
					id = tt.id,
					sourceType = tt.sourceType,
					sourceNumber = tt.sourceNumber,
					sourcePortChannel = tt.sourcePortChannel,
					sourceIP = tt.sourceIP,
					sourceState = tt.sourceState,
					sourceStyle = tt.sourceStyle,
					sourceTransportMode = tt.sourceTransportMode,
					sourceVlanList = tt.sourceVlanList,
					name = tt.name,
					type = tt.type,
					number = tt.number,
					portChannel = tt.portChannel,
					linkType = tt.linkType,
					networks = tt.networks,
					lastUpdateSystem = tt.lastUpdateSystem,
					lastUpdateDate = tt.lastUpdateDate,
					info = tt.info.Replace("<", "(").Replace(">", ")")
				}));
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetLinksByDevice():790");
		}
		return new JsonResult(null);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<IActionResult> GetDownloadLinks([FromForm] GridParam gridParam)
	{
		try
		{
			Devices device = MyDevices.Where((Devices tt) => tt.id == ObjectId.Parse(gridParam.id)).SingleOrDefault();
			string text = device.configFile.Replace("-Running", string.Empty).Replace("-Startup", string.Empty).Replace("_Running", string.Empty)
				.Replace("_Startup", string.Empty);
			string deviceName = device.name;
			if (deviceName.Contains("["))
			{
				deviceName = deviceName.Substring(0, deviceName.IndexOf("["));
			}
			List<ConfigFiles> list = new List<ConfigFiles>();
			if (!string.IsNullOrEmpty(text))
			{
				list = await _configFileService.FindConfigAsync(text);
			}
			if (list == null || list.Count == 0)
			{
				list = await _configFileService.FindAsync(deviceName);
			}
			if (list != null)
			{
				return new JsonResult(list.Select((ConfigFiles tt) => new
				{
					id = tt.id,
					address = tt.address.Replace(_appSettings.Value.ftpConfig.LocalPath, string.Empty) + ".txt",
					lastDate = tt.lastModified.ToString("yyyy/MM/dd HH:mm"),
					link = base.Url.Action("DownloadConfig", "CMDB", new
					{
						devId = device.id,
						id = tt.id
					})
				}));
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetDownloadLinks():816");
		}
		return new JsonResult(null);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public IActionResult GetServicesByDevice([FromForm] GridParam gridParam)
	{
		JqGridResponse jqGridResponse = new JqGridResponse();
		try
		{
			List<Services> servicesByDeviceID = GetServicesByDeviceID(ObjectId.Parse(gridParam.id));
			if (servicesByDeviceID != null)
			{
				jqGridResponse = new JqGridResponse
				{
					TotalPagesCount = 1,
					PageIndex = 0,
					TotalRecordsCount = servicesByDeviceID.Count()
				};
				foreach (Services item in servicesByDeviceID)
				{
					jqGridResponse.Records.Add(new JqGridRecord(Convert.ToString(item.id), new Services
					{
						id = item.id,
						code = item.code,
						nameFa = item.nameFa,
						nameEn = item.nameEn,
						view_networks = item.view_networks,
						view_customer_count = item.view_customer_count,
						info = item.info
					}));
				}
				jqGridResponse.Reader.RepeatItems = false;
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetServicesByDevice():889");
		}
		return new JqGridJsonResult(jqGridResponse);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<JsonResult> GetPatchs()
	{
		await AddEventLogAsync("CMDB-Show-Patchs", base.CurrentUser.mainUserInfo.UserName);
		try
		{
			List<Devices> list = MyDevices.Where((Devices tt) => tt.view_patch_count > 0).ToList();
			if (list != null)
			{
				List<object> list2 = new List<object>();
				foreach (Devices item in list)
				{
					foreach (PatchDevice patch in item.patchs)
					{
						list2.Add(new
						{
							id = $"{item.id}-{patch.id.ToString()}",
							deviceId = item.id.ToString(),
							device = item.name,
							type = item.type,
							network = item.tag.network,
							riskPriority = ((patch.riskPriority == PriorityLevel.Top) ? "بالا" : ((patch.riskPriority == PriorityLevel.Medium) ? "متوسط" : "پایین")),
							riskFactor = patch.riskFactor,
							requireVersion = patch.requireVersion,
							updateStatus = ((patch.updateStatus == PatchUpdateStatus.doing) ? "در دست اقدام" : ((patch.updateStatus == PatchUpdateStatus.ok) ? "انجام شده" : "انجام نشده")),
							date = ((patch.date.Year > 2000) ? patch.date.ToString("yyyy/MM/dd") : ""),
							descriptionType = patch.descriptionType,
							description = patch.description,
							lastUpdate = patch.lastUpdate.date.ToString("yyyy/MM/dd"),
							lastUserActivity = patch.lastUpdate.user.name
						});
					}
				}
				return new JsonResult(list2);
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetPatchs():617");
		}
		return new JsonResult(null);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public IActionResult GetPatchsByDevice([FromForm] GridParam gridParam)
	{
		try
		{
			List<PatchDevice> patchsByDeviceID = GetPatchsByDeviceID(ObjectId.Parse(gridParam.id));
			if (patchsByDeviceID != null)
			{
				return new JsonResult(patchsByDeviceID.Select((PatchDevice patch) => new PatchDeviceInput
				{
					id = patch.id.ToString(),
					deviceId = gridParam.id,
					riskPriorityId = (byte)patch.riskPriority,
					riskPriority = ((patch.riskPriority == PriorityLevel.Top) ? "بالا" : ((patch.riskPriority == PriorityLevel.Medium) ? "متوسط" : "پایین")),
					riskFactorId = patch.riskFactorId,
					riskFactor = patch.riskFactor,
					requireVersion = patch.requireVersion,
					updateStatusId = (byte)patch.updateStatus,
					updateStatus = ((patch.updateStatus == PatchUpdateStatus.doing) ? "در دست اقدام" : ((patch.updateStatus == PatchUpdateStatus.ok) ? "انجام شده" : "انجام نشده")),
					date = ((patch.date.Year > 2000) ? patch.date.ToString("yyyy/MM/dd") : ""),
					descriptionTypeId = patch.descriptionTypeId,
					descriptionType = patch.descriptionType,
					description = patch.description,
					lastUpdate = patch.lastUpdate.date.ToString("yyyy/MM/dd"),
					lastUserActivity = patch.lastUpdate.user.name
				}));
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetPatchsByDevice():671");
		}
		return new JsonResult(null);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<ResultOptions> DeletePatch([FromForm] PatchDeviceInput model)
	{
		_ = 5;
		try
		{
			List<string> networkAccess = (from tt in base._accessUser.networks
				where tt.access == "write"
				select tt into a
				select a.network).ToList();
			PublicJsonResult publicJsonResult = await _deviceService.DeletePatchAsync(model, networkAccess);
			switch (publicJsonResult.Status)
			{
			case 1:
			{
				Devices device = (Devices)publicJsonResult.Result;
				await AddEventLogAsync("CMDB-Delete-Patch", "Device: " + device.name);
				Devices devices = _cachProvider._AllDevices.FirstOrDefault((Devices tt) => tt.id == device.id);
				devices.patchs = device.patchs;
				devices.view_patch_count = device.patchs.Count;
				await GetResultOkAsync("successDelete", "اطلاعات با موفقیت حذف گردید !");
				break;
			}
			case -1:
				await GetResultErrorGridAsync("device.invalidId", "اطلاعات تجهیز اشتباه ارسال شده است!");
				break;
			case -2:
				await GetResultErrorGridAsync("device.invalidPatchId", "اطلاعات وصله امنیتی اشتباه ارسال شده است!");
				break;
			case -3:
				await GetResultErrorGridAsync("cmdb.notAccessUpdate", "شما دسترسی لازم برای بروز رسانی اطلاعات تجهیزات این شبکه را ندارید!");
				break;
			default:
				GetResultServerGridError("خطای سمت سرور پیش آمده! " + publicJsonResult.Message);
				break;
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "DeletePatch():971");
			base.Response.StatusCode = 300;
			resultOption.message = ex.Message;
		}
		return resultOption;
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<ResultOptions> EditPatch([FromForm] PatchDeviceInput model)
	{
		_ = 8;
		try
		{
			List<string> networkAccess = (from tt in base._accessUser.networks
				where tt.access == "write"
				select tt into a
				select a.network).ToList();
			PublicJsonResult result = await _deviceService.AddPatchAsync(model, GetCurrentUserInfo(), networkAccess);
			Devices device;
			switch (result.Status)
			{
			case 1:
			{
				device = (Devices)result.Result;
				await AddEventLogAsync("CMDB-Add-Patch", "Device: " + device.name);
				Devices devices = _cachProvider._AllDevices.FirstOrDefault((Devices tt) => tt.id == device.id);
				devices.patchs = device.patchs;
				devices.view_patch_count = device.patchs.Count;
				resultOption.htmlString1 = result.Message;
				resultOption.htmlString2 = GetCurrentUserInfo().name;
				resultOption.htmlString3 = DateTime.Now.ToString("yyyy/MM/dd");
				await GetResultOkAsync("addSuccess", "اطلاعات با موفقیت به سامانه اضافه گردید.");
				break;
			}
			case 2:
				device = (Devices)result.Result;
				await AddEventLogAsync("CMDB-Edit-Patch", "Device: " + device.name);
				resultOption.htmlString1 = result.Message;
				resultOption.htmlString2 = GetCurrentUserInfo().name;
				resultOption.htmlString3 = DateTime.Now.ToString("yyyy/MM/dd");
				_cachProvider._AllDevices.FirstOrDefault((Devices tt) => tt.id == device.id).patchs = device.patchs;
				await GetResultOkAsync("updateSuccess", "اطلاعات با موفقیت در سامانه ویرایش گردید.");
				break;
			case -1:
				await GetResultErrorAsync("device.invalidId", "اطلاعات تجهیز اشتباه ارسال شده است!");
				break;
			case -2:
				await GetResultErrorAsync("device.invalidPatchId", "اطلاعات وصله امنیتی اشتباه ارسال شده است!");
				break;
			case -3:
				await GetResultErrorAsync("cmdb.notAccessUpdate", "شما دسترسی لازم برای بروز رسانی اطلاعات تجهیزات این شبکه را ندارید!");
				break;
			default:
				await GetResultErrorAsync("خطای سمت سرور پیش آمده! " + result.Message);
				break;
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "EditPatch():745");
			resultOption.status = -1;
			resultOption.message = ex.Message;
		}
		return resultOption;
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<ResultOptions> UpdateDeviceDetails([FromForm] DeviceInputModel model)
	{
		_ = 5;
		try
		{
			List<string> networkAccess = (from tt in base._accessUser.networks
				where tt.access == "write"
				select tt into a
				select a.network).ToList();
			PublicJsonResult result = await _deviceService.UpdateDeviceAsync(model, GetCurrentUserInfo(), networkAccess);
			switch (result.Status)
			{
			case 1:
			{
				Devices device = (Devices)result.Result;
				await AddEventLogAsync("CMDB-Update-Device-Level", "Device: " + device.name);
				Devices devices = _cachProvider._AllDevices.FirstOrDefault((Devices tt) => tt.id == device.id);
				devices.tag.level = device.tag.level;
				devices.gridview_lastUpdateDate = DateTime.Now.ToString("yyyy/MM/dd");
				devices.gridview_networkLevel = device.tag.level;
				resultOption.htmlString1 = result.Message;
				resultOption.htmlString2 = GetCurrentUserInfo().name;
				resultOption.htmlString3 = DateTime.Now.ToString("yyyy/MM/dd");
				await GetResultOkAsync("updateSuccess", "اطلاعات با موفقیت در سامانه بروزرسانی گردید.");
				break;
			}
			case -1:
				await GetResultErrorAsync("device.invalidId", "اطلاعات تجهیز اشتباه ارسال شده است!");
				break;
			case -2:
				await GetResultErrorAsync("cmdb.notAccessUpdate", "شما دسترسی لازم برای بروز رسانی اطلاعات تجهیزات این شبکه را ندارید!");
				break;
			default:
				await GetResultErrorAsync("خطای سمت سرور پیش آمده! " + result.Message);
				break;
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "UpdateDeviceDetails():818");
			resultOption.status = -1;
			resultOption.message = ex.Message;
		}
		return resultOption;
	}

	[Authorize(Roles = "Admin")]
	[BreadCrumb(Title = "Devices", Order = 2, GlyphIcon = "icon icon-database ")]
	[AddHeader("Devices")]
	public async Task<IActionResult> Devices1()
	{
		base.ViewData["AccessEdit"] = false;
		await SetDevicePatchParam();
		return View();
	}

	[HttpPost]
	[Authorize(Roles = "Admin")]
	public async Task<IActionResult> GetDevices1(JqGridRequest request, string deviceId)
	{
		JqGridResponse response = new JqGridResponse();
		try
		{
			await AddEventLogAsync("CMDB-Show-Devices_old", base.CurrentUser.mainUserInfo.UserName);
			response = GetDevicesExport(request, deviceId);
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetDevices1():848");
		}
		return new JqGridJsonResult(response);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public IActionResult GetDevicesBySearch(JqGridRequest request, string deviceId)
	{
		JqGridResponse jqGridResponse = new JqGridResponse();
		try
		{
			if (request.SearchingFilters == null || (request.SearchingFilters.Filters.Count == 0 && request.SearchingFilters.Groups.Count == 0))
			{
				jqGridResponse = new JqGridResponse
				{
					TotalPagesCount = 0,
					PageIndex = request.PageIndex,
					TotalRecordsCount = 0
				};
				jqGridResponse.Reader.RepeatItems = false;
			}
			else
			{
				jqGridResponse = GetDevicesExport(request, deviceId);
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetDevices():2144");
		}
		return new JqGridJsonResult(jqGridResponse);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public IActionResult GetDevicesByService(JqGridRequest request, string serviceId)
	{
		JqGridResponse jqGridResponse = new JqGridResponse();
		try
		{
			List<Devices> devicesByServiceID = GetDevicesByServiceID(ObjectId.Parse(serviceId));
			if (devicesByServiceID != null)
			{
				int totalRecordsCount = devicesByServiceID.Count();
				jqGridResponse = new JqGridResponse
				{
					TotalPagesCount = 1,
					PageIndex = request.PageIndex,
					TotalRecordsCount = totalRecordsCount
				};
				foreach (Devices item in devicesByServiceID)
				{
					jqGridResponse.Records.Add(new JqGridRecord(Convert.ToString(item.id), new Devices
					{
						id = item.id,
						name = item.name,
						type = item.type,
						ipManagement = item.ipManagement,
						series = item.series,
						model = item.model,
						view_devint_count = item.view_devint_count,
						view_patch_count = item.view_patch_count,
						view_locations = item.view_locations,
						backupDevice = item.backupDevice,
						powerBackup = item.powerBackup,
						isVirtual = item.isVirtual,
						localUsername = item.localUsername,
						ownerGroup = item.ownerGroup
					}));
				}
				jqGridResponse.Reader.RepeatItems = false;
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetDevicesByService():388");
		}
		return new JqGridJsonResult(jqGridResponse);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public IActionResult GetInterfacesByDevice([FromForm] GridParam gridParam)
	{
		JqGridResponse jqGridResponse = new JqGridResponse();
		try
		{
			List<Devint> interfacesByDeviceID = GetInterfacesByDeviceID(ObjectId.Parse(gridParam.id));
			if (interfacesByDeviceID != null)
			{
				jqGridResponse = new JqGridResponse
				{
					TotalPagesCount = 1,
					PageIndex = 0,
					TotalRecordsCount = interfacesByDeviceID.Count()
				};
				foreach (Devint item in interfacesByDeviceID)
				{
					jqGridResponse.Records.Add(new JqGridRecord(Convert.ToString(item.id), new Devint
					{
						id = item.id,
						type = item.type,
						number = item.number,
						portChannel = item.portChannel,
						ip = (string.IsNullOrEmpty(item.ip) ? string.Empty : item.ip),
						transportMode = (string.IsNullOrEmpty(item.transportMode) ? string.Empty : item.transportMode),
						info = item.info
					}));
				}
				jqGridResponse.Reader.RepeatItems = false;
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetInterfacesByDevice():428");
		}
		return new JqGridJsonResult(jqGridResponse);
	}

	internal JqGridResponse GetDevicesExport(JqGridRequest request, string deviceId)
	{
		JqGridResponse jqGridResponse = new JqGridResponse();
		IQueryable<Devices> query = (string.IsNullOrEmpty(deviceId) ? MyDevices : MyDevices.Where((Devices tt) => tt.id == ObjectId.Parse(deviceId)));
		query = new JqGridSearch().ApplyFilter(query, request);
		string arg = (string.IsNullOrEmpty(request.SortingName) ? "id" : request.SortingName);
		arg = $"{arg} {request.SortingOrder.ToString()}";
		List<Devices> list = query.OrderBy(arg).Skip(request.PageIndex * request.RecordsCount).Take(request.PagesCount * request.RecordsCount)
			.ToList();
		int num = query.Count();
		jqGridResponse = new JqGridResponse
		{
			TotalPagesCount = (int)Math.Ceiling((float)num / (float)request.RecordsCount),
			PageIndex = request.PageIndex,
			TotalRecordsCount = num
		};
		foreach (Devices item in list)
		{
			jqGridResponse.Records.Add(new JqGridRecord(Convert.ToString(item.id), new Devices
			{
				id = item.id,
				name = item.name,
				type = item.type,
				gridview_network = item.tag.network,
				view_tags = item.view_tags,
				ipManagement = item.ipManagement,
				series = item.series,
				model = item.model,
				view_devint_count = item.view_devint_count,
				view_patch_count = item.view_patch_count,
				site = item.site,
				subSite = item.subSite,
				view_locations = item.view_locations,
				gridview_networkLevel = item.tag.level,
				osVersion = item.osVersion,
				view_ACSServerIPs = item.view_ACSServerIPs,
				label = item.label,
				propertyNo = item.propertyNo,
				gridview_logSourceIP = item.log.sourceIP,
				view_logServers = item.view_logServers,
				gridview_logSeverity = item.log.severity,
				gridview_ntpConfigSyncType = item.ntpConfig.syncType,
				view_ntpConfigIPs = item.view_ntpConfigIPs,
				backupDevice = item.backupDevice,
				powerBackup = item.powerBackup,
				isVirtual = item.isVirtual,
				localUsername = item.localUsername,
				ownerGroup = item.ownerGroup,
				gridview_lastUpdateName = item.lastUpdate.user.name,
				gridview_lastUpdateSystem = item.lastUpdate.system,
				gridview_lastUpdateDate = item.lastUpdate.date.ToString("yyyy-MM-dd")
			}));
		}
		jqGridResponse.Reader.RepeatItems = false;
		return jqGridResponse;
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public IActionResult GetPatchsByDevice1([FromForm] GridParam gridParam)
	{
		JqGridResponse jqGridResponse = new JqGridResponse();
		try
		{
			List<PatchDevice> patchsByDeviceID = GetPatchsByDeviceID(ObjectId.Parse(gridParam.id));
			if (patchsByDeviceID != null)
			{
				jqGridResponse = new JqGridResponse
				{
					TotalPagesCount = 1,
					PageIndex = 0,
					TotalRecordsCount = patchsByDeviceID.Count()
				};
				foreach (PatchDevice item in patchsByDeviceID)
				{
					jqGridResponse.Records.Add(new JqGridRecord(Convert.ToString(item.id), new PatchDeviceInput
					{
						id = $"{gridParam.id}-{item.id.ToString()}",
						deviceId = gridParam.id,
						riskPriorityId = (byte)item.riskPriority,
						riskPriority = ((item.riskPriority == PriorityLevel.Top) ? "بالا" : ((item.riskPriority == PriorityLevel.Medium) ? "متوسط" : "پایین")),
						riskFactorId = item.riskFactorId,
						riskFactor = item.riskFactor,
						requireVersion = item.requireVersion,
						updateStatusId = (byte)item.updateStatus,
						updateStatus = ((item.updateStatus == PatchUpdateStatus.doing) ? "در دست اقدام" : ((item.updateStatus == PatchUpdateStatus.ok) ? "انجام شده" : "انجام نشده")),
						date = ((item.date.Year > 2000) ? item.date.ToString("yyyy/MM/dd") : ""),
						descriptionTypeId = item.descriptionTypeId,
						descriptionType = item.descriptionType,
						description = item.description
					}));
				}
				jqGridResponse.Reader.RepeatItems = false;
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetPatchsByDevice():935");
		}
		return new JqGridJsonResult(jqGridResponse);
	}

	internal async Task SetDevicePatchParam()
	{
		List<InternalDatas> source = await _internalDataService.GetAllAsync();
		List<InternalDatas> list = source.Where((InternalDatas tt) => tt.Category == "riskFactor").ToList();
		List<InternalDatas> list2 = source.Where((InternalDatas tt) => tt.Category == "descriptionType").ToList();
		string text = string.Empty;
		foreach (InternalDatas item in list)
		{
			text += $"{item.IntValue}:{item.TextValue};";
		}
		text = text.Substring(0, text.Length - 1);
		string text2 = string.Empty;
		foreach (InternalDatas item2 in list2)
		{
			text2 += $"{item2.IntValue}:{item2.TextValue};";
		}
		text2 = text2.Substring(0, text2.Length - 1);
		base.ViewData["RiskFactorItems"] = text;
		base.ViewData["DescriptionTypeItems"] = text2;
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<IActionResult> GetVlans([FromForm] GridParam gridParam)
	{
		try
		{
			Devices devices = MyDevices.Where((Devices tt) => tt.id == ObjectId.Parse(gridParam.id)).SingleOrDefault();
			List<string> value = (from tt in devices.vLans
				where tt.name.Length > 0 && tt.active
				select tt.name).ToList();
			return new JsonResult(value);
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetDownloadLinks():816");
		}
		return new JsonResult(null);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<ResultOptions> EditDeviceService([FromForm] DeviceInputModel model)
	{
		_ = 5;
		try
		{
			List<string> networkAccess = (from tt in base._accessUser.networks
				where tt.access == "write"
				select tt into a
				select a.network).ToList();
			PublicJsonResult result = await _deviceService.UpdateDeviceServicesAsync(model, GetCurrentUserInfo(), networkAccess);
			switch (result.Status)
			{
			case 1:
			{
				Devices device = (Devices)result.Result;
				await AddEventLogAsync("CMDB-Update-Device-Services", "Device: " + device.name + " | services : " + model.services);
				Devices devices = _cachProvider._AllDevices.FirstOrDefault((Devices tt) => tt.id == device.id);
				devices.services = device.services;
				devices.gridview_lastUpdateDate = DateTime.Now.ToString("yyyy/MM/dd");
				devices.view_services = model.services.Replace("\"", "").Replace("[", "").Replace("]", "");
				resultOption.htmlString1 = result.Message;
				resultOption.htmlString2 = GetCurrentUserInfo().name;
				resultOption.htmlString3 = DateTime.Now.ToString("yyyy/MM/dd");
				await GetResultOkAsync("updateSuccess", "اطلاعات با موفقیت در سامانه بروزرسانی گردید.");
				break;
			}
			case -1:
				await GetResultErrorAsync("device.invalidId", "اطلاعات تجهیز اشتباه ارسال شده است!");
				break;
			case -2:
				await GetResultErrorAsync("cmdb.notAccessUpdate", "شما دسترسی لازم برای بروز رسانی اطلاعات تجهیزات این شبکه را ندارید!");
				break;
			default:
				await GetResultErrorAsync("خطای سمت سرور پیش آمده! " + result.Message);
				break;
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "EditDeviceService():1043");
			base.Response.StatusCode = 300;
			resultOption.message = ex.Message;
		}
		return resultOption;
	}

	[HttpPost]
	[Authorize(Roles = "Admin")]
	public async Task<ResultOptions> EditService([FromForm] ServiceViewParam item)
	{
		try
		{
			if (!base.User.IsInRole("Admin"))
			{
				await GetResultErrorAsync("service.accessEdit", "در حال حاضر فقط برای ادمین سامانه دسترسی ویرایش سرویس ها تعریف شده است!");
			}
			else
			{
				item.info = ((item.info == null) ? string.Empty : item.info);
				if (string.IsNullOrEmpty(item.nameFa) || string.IsNullOrEmpty(item.nameEn))
				{
					await GetResultErrorAsync("service.forceName", "نام و نام لاتین و شبکه را باید حتما تکمیل فرمایید !");
				}
				else if (item.nameFa.HasScriptText() || item.nameEn.HasScriptText() || item.info.HasScriptText())
				{
					await GetResultErrorAsync("service.security", "اطلاعات خواسته شده نمی توانند شامل اسکریپت اجرایی باشند !");
				}
				else if (item.id == null || item.id == string.Empty)
				{
					if (_cachProvider._AllServices.Where((Services tt) => tt.nameEn == item.nameEn).Count() > 0)
					{
						await GetResultErrorAsync("service.exist", "سرویس دیگری با همین نام/ کد در سامانه وجود دارد !");
					}
					else
					{
						Services service = new Services
						{
							nameFa = item.nameFa.ImproveText().ToSecurText(),
							nameEn = item.nameEn.ToSecurText(),
							info = item.info.ImproveText().ToSecurText()
						};
						await _serviceService.CreateAsync(service);
						service.view_networks = string.Empty;
						service.view_customer_count = 0;
						_cachProvider._AllServices.ToList().Add(service);
						await GetResultOkAsync("service.ok", "سرویس جدید با موفقیت به سامانه اضافه شد !");
					}
				}
				else
				{
					ObjectId id = ObjectId.Parse(item.id);
					Services services;
					Services service = (services = _cachProvider._AllServices.FirstOrDefault((Services tt) => tt.id == id));
					if (services == null)
					{
						await GetResultErrorAsync("service.access", "رکورد به صورت صحیح انتخاب نشده است یا شما دسترسی برای تغییر آن را ندارید!");
					}
					else if (_cachProvider._AllServices.FirstOrDefault((Services tt) => tt.id != id && tt.nameEn == item.nameEn) != null)
					{
						await GetResultErrorAsync("service.exist", "سرویس دیگری با همین نام/ کد در سامانه وجود دارد!");
					}
					else
					{
						bool flag = false;
						foreach (string net in service.networks)
						{
							AccessNetwork accessNetwork = base._accessUser.networks.Where((AccessNetwork tt) => tt.network == net && tt.access == "write").FirstOrDefault();
							if (accessNetwork != null)
							{
								flag = true;
							}
						}
						if (!flag)
						{
							await GetResultErrorAsync("service.accessUser", "شما برای این کار دسترسی ندارید!");
						}
						else
						{
							service.nameFa = item.nameFa.ImproveText().ToSecurText();
							service.nameEn = item.nameEn.ToSecurText();
							service.info = item.info.ImproveText().ToSecurText();
							await _serviceService.UpdateAsync(service);
							await GetResultOkAsync("service.ok", "سرویس با موفقیت ویرایش شد !");
						}
					}
				}
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "EditService():4126");
			base.Response.StatusCode = 300;
			resultOption.message = ex.Message;
		}
		return resultOption;
	}

	internal List<Devices> GetDevicesByServiceID(ObjectId Id)
	{
		IMongoCollection<Links> collection = _mongoDbContext.GetCollection<Links>("Links");
		List<Links> source = (from tt in collection.AsQueryable()
			where tt.customerServices.Any((CustomerService a) => a.services.Any((IDName b) => b.id == Id))
			select tt).ToList();
		List<ObjectId> LinkSide1 = source.Select((Links tt) => tt.side1).ToList();
		List<ObjectId> list = source.Select((Links tt) => tt.side2).ToList();
		foreach (ObjectId item in list)
		{
			if (!LinkSide1.Contains(item))
			{
				LinkSide1.Add(item);
			}
		}
		IQueryable<Devices> queryable = from tt in MyDevices
			where tt.devint.Any((Devint a) => LinkSide1.Contains(a.id))
			select tt into t
			select (t);
		if (queryable != null)
		{
			return queryable.ToList();
		}
		return new List<Devices>();
	}

	internal List<Services> GetServicesByCustomerID(ObjectId Id)
	{
		IEnumerable<Services> enumerable = from tt in _cachProvider._AllServices
			where tt.customers.Any((IDName t) => t.id == Id)
			select tt into t
			select (t);
		if (enumerable != null)
		{
			return enumerable.ToList();
		}
		return new List<Services>();
	}

	internal Services GetServiceByServiceID(ObjectId Id)
	{
		Services services = _cachProvider._AllServices.Where((Services tt) => tt.id == Id).FirstOrDefault();
		if (services != null)
		{
			return services;
		}
		return new Services();
	}

	internal List<Services> GetServicesByDeviceID(ObjectId Id)
	{
		Devices devices = _cachProvider._AllDevices.Where((Devices tt) => tt.id == Id).SingleOrDefault();
		List<ObjectId> serviceIDs = new List<ObjectId>();
		if (devices.devint != null && devices.devint.Count > 0)
		{
			List<ObjectId> devintIDs = devices.devint.Select((Devint tt) => tt.id).ToList();
			IMongoCollection<Links> collection = _mongoDbContext.GetCollection<Links>("Links");
			List<Links> list = (from tt in collection.AsQueryable()
				where devintIDs.Contains(tt.side1) || devintIDs.Contains(tt.side2)
				select tt).ToList();
			foreach (Links item in list)
			{
				foreach (CustomerService customerService in item.customerServices)
				{
					foreach (IDName service in customerService.services)
					{
						if (!serviceIDs.Contains(service.id))
						{
							serviceIDs.Add(service.id);
						}
					}
				}
			}
			IEnumerable<Services> enumerable = from tt in _cachProvider._AllServices
				where serviceIDs.Contains(tt.id)
				select (tt);
			if (enumerable != null)
			{
				return enumerable.ToList();
			}
			return new List<Services>();
		}
		return new List<Services>();
	}

	internal List<PatchDevice> GetPatchsByDeviceID(ObjectId Id)
	{
		Devices devices = _cachProvider._AllDevices.Where((Devices tt) => tt.id == Id).SingleOrDefault();
		if (devices != null && devices.patchs != null && devices.view_patch_count > 0)
		{
			AddEventLogAsync("CMDB-Show-Patchs", "Device: " + devices.name).Wait();
			return devices.patchs;
		}
		return new List<PatchDevice>();
	}

	internal List<Devint> GetInterfacesByDeviceID(ObjectId Id)
	{
		Devices devices = _cachProvider._AllDevices.Where((Devices tt) => tt.id == Id).SingleOrDefault();
		if (devices != null && devices.devint != null && devices.devint.Count > 0)
		{
			AddEventLogAsync("CMDB-Show-Links-Device", "Device: " + devices.name).Wait();
			return devices.devint;
		}
		return new List<Devint>();
	}

	internal List<DeviceLink> GetLinkInterface(Devint item, List<Links> allLinks)
	{
		DeviceLink deviceLink = null;
		List<DeviceLink> list = new List<DeviceLink>();
		List<Links> list2 = allLinks.Where((Links tt) => tt.side1 == item.id).ToList();
		List<Links> list3 = allLinks.Where((Links tt) => tt.side2 == item.id).ToList();
		bool flag = false;
		if (list2 != null && list2.Count > 0)
		{
			foreach (Links link in list2)
			{
				deviceLink = (from tt in _cachProvider._AllDevices.AsQueryable()
					where tt.devint.Any((Devint t) => link.side2 == t.id)
					select tt into b
					select new DeviceLink
					{
						id = link.id,
						name = b.name,
						number = b.devint.Where((Devint c) => c.id == link.side2).SingleOrDefault().number,
						portChannel = b.devint.Where((Devint c) => c.id == link.side2).SingleOrDefault().portChannel,
						type = b.devint.Where((Devint c) => c.id == link.side2).SingleOrDefault().type,
						linkType = link.linkType,
						networks = string.Join(", ", link.networks.ToArray()),
						info = link.info,
						lastUpdateDate = link.lastUpdate.date.ToString("yyyy-MM-dd"),
						lastUpdateSystem = link.lastUpdate.system,
						lastUpdateName = link.lastUpdate.user.name,
						sourceId = item.id,
						sourceType = item.type,
						sourceNumber = item.number,
						sourcePortChannel = item.portChannel,
						sourceIP = (string.IsNullOrEmpty(item.ip) ? string.Empty : item.ip),
						sourceState = item.state,
						sourceStyle = item.style,
						sourceTransportMode = (string.IsNullOrEmpty(item.transportMode) ? string.Empty : item.transportMode),
						sourceVlanList = ((item.vlanList != null) ? string.Join(",", item.vlanList.ToArray()) : string.Empty)
					}).SingleOrDefault();
				if (deviceLink != null)
				{
					list.Add(deviceLink);
					flag = true;
				}
			}
		}
		if (list3 != null && list3.Count > 0)
		{
			foreach (Links link in list3)
			{
				deviceLink = (from tt in _cachProvider._AllDevices
					where tt.devint.Any((Devint t) => link.side1 == t.id)
					select tt into b
					select new DeviceLink
					{
						id = link.id,
						name = b.name,
						number = b.devint.Where((Devint c) => c.id == link.side1).FirstOrDefault().number,
						portChannel = b.devint.Where((Devint c) => c.id == link.side1).FirstOrDefault().portChannel,
						type = b.devint.Where((Devint c) => c.id == link.side1).FirstOrDefault().type,
						linkType = link.linkType,
						networks = string.Join(", ", link.networks.ToArray()),
						info = link.info,
						lastUpdateDate = link.lastUpdate.date.ToString("yyyy-MM-dd"),
						lastUpdateSystem = link.lastUpdate.system,
						lastUpdateName = link.lastUpdate.user.name,
						sourceId = item.id,
						sourceType = item.type,
						sourceNumber = item.number,
						sourcePortChannel = item.portChannel,
						sourceIP = (string.IsNullOrEmpty(item.ip) ? string.Empty : item.ip),
						sourceState = item.state,
						sourceStyle = item.style,
						sourceTransportMode = (string.IsNullOrEmpty(item.transportMode) ? string.Empty : item.transportMode),
						sourceVlanList = ((item.vlanList != null) ? string.Join(",", item.vlanList.ToArray()) : string.Empty)
					}).FirstOrDefault();
				if (deviceLink != null)
				{
					list.Add(deviceLink);
					flag = true;
				}
			}
		}
		if (!flag)
		{
			deviceLink = new DeviceLink();
			deviceLink.id = ObjectId.GenerateNewId();
			deviceLink.type = "-1";
			deviceLink.name = "UnKnown";
			deviceLink.sourceId = item.id;
			deviceLink.sourceType = item.type;
			deviceLink.sourceNumber = item.number;
			deviceLink.sourceState = item.state;
			deviceLink.sourceStyle = item.style;
			deviceLink.sourcePortChannel = item.portChannel;
			deviceLink.sourceIP = (string.IsNullOrEmpty(item.ip) ? string.Empty : item.ip);
			deviceLink.sourceTransportMode = (string.IsNullOrEmpty(item.transportMode) ? string.Empty : item.transportMode);
			deviceLink.sourceVlanList = ((item.vlanList != null) ? string.Join(",", item.vlanList.ToArray()) : string.Empty);
			list.Add(deviceLink);
		}
		return list;
	}

	[BreadCrumb(Title = "IPAM", Order = 2, GlyphIcon = "icon icon-network2 ")]
	[AddHeader("IPAM")]
	[Authorize(Roles = "CMDB")]
	public ActionResult IPAM()
	{
		List<string> accessNetworks = GetAccessEditIpamNetworkList();
		base.ViewBag.AccessNetwork = accessNetworks;
		base.ViewBag.IpamMainSite = (from tt in _cachProvider._AllSites
			group tt by tt.main into tt
			select tt.Key).ToList();
		base.ViewBag.IpamServices = (from tt in MyIPServices
			where accessNetworks.Any((string a) => a == tt.network)
			select tt.nameEn).ToList();
		base.ViewBag.IpamTags = ipTags;
		return View();
	}

	[BreadCrumb(Title = "IPAMServices", Order = 2, GlyphIcon = "icon icon-swap-horizontal ")]
	[AddHeader("IPAMServices")]
	[Authorize(Roles = "CMDB")]
	public ActionResult IPAMServices()
	{
		base.ViewBag.AccessNetwork = GetAccessEditIpamNetworkList();
		return View();
	}

	[BreadCrumb(Title = "IPAMUpload", Order = 2, GlyphIcon = "icon icon-cloud-upload ")]
	[AddHeader("IPAMUpload")]
	[Authorize(Roles = "CMDB")]
	public ActionResult IPAMUpload()
	{
		List<string> accessEditIpamNetworkList = GetAccessEditIpamNetworkList();
		if (accessEditIpamNetworkList.Count == 0)
		{
			return RedirectToAction("IPAM", "CMDB");
		}
		return View();
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<JsonResult> GetIPList()
	{
		try
		{
			await AddEventLogAsync("CMDB-Show-IPAM", base.CurrentUser.mainUserInfo.UserName);
			return new JsonResult(MyIPAM.Select((IPLists ip) => new IPAMOutput(ip)).ToList());
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetIPList():67");
		}
		return new JsonResult(new GetResponseParams());
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<JsonResult> GetIPServices()
	{
		try
		{
			await AddEventLogAsync("CMDB-Show-IPAM-Service", base.CurrentUser.mainUserInfo.UserName);
			return new JsonResult(_cachProvider._AllIPServices.Select((IPServices ipService) => new IPServiceInput(ipService)).ToList());
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetIPServices():195");
		}
		return new JsonResult(new GetResponseParams());
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<JsonResult> GetIPDetails([FromForm] GridParam gridParam)
	{
		try
		{
			IPLists localIPListByID = GetLocalIPListByID(gridParam.id);
			if (localIPListByID != null)
			{
				return new JsonResult(from tt in localIPListByID.devices
					where !tt.isEdge
					select new IPAMDetailsInput
					{
						id = tt.id.ToString(),
						parentId = gridParam.id,
						name = tt.name,
						ipAddress = tt.ipAddress,
						type = tt.type,
						dnsName = tt.dnsName,
						status = tt.status,
						serviceIP1 = tt.serviceIP1,
						serviceIP2 = tt.serviceIP2,
						vsIP = tt.VSIP,
						interSiteIP = tt.interSiteIP,
						externalSRCIP = tt.externalSRCIP,
						description = tt.description
					});
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetIPDetails():137");
		}
		return new JsonResult(null);
	}

	[HttpPost]
	[Authorize(Roles = "IPAMWrite")]
	public async Task<JsonResult> GetSiteSecondary([FromForm] GridParam gridParam)
	{
		try
		{
			if (string.IsNullOrEmpty(gridParam.id))
			{
				await GetResultErrorAsync("cmdb.nullSiteSecondary", "نام سایت اصلی برای دریافت لیست سایت ثانویه وارد نشده است!");
			}
			else
			{
				List<string> list = new List<string>();
				List<Sites> list2 = _cachProvider._AllSites.Where((Sites tt) => tt.main == gridParam.id).ToList();
				foreach (Sites item in list2)
				{
					list.Add(item.secondary);
				}
				resultOption.status = 1;
				resultOption.result = list;
			}
		}
		catch (Exception ex)
		{
			resultOption.message = ex.Message;
			_logger.LogCritical(ex, "GetSiteSecondary():224");
		}
		return Json(resultOption);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public async Task<JsonResult> GetSearchDevice([FromForm] SearchIPAMModel model)
	{
		GetResponseParams result = new GetResponseParams();
		try
		{
			if (!string.IsNullOrEmpty(model.value))
			{
				await AddEventLogAsync("CMDB-Search-IPAM", base.CurrentUser.mainUserInfo.UserName, model.value);
				IQueryable<IPLists> source = MyIPAM.Where((IPLists x) => x.devices.Exists((IPListItem tt) => (model.filterIpAddress ? tt.ipAddress.Contains(model.value) : false) || (model.filterName ? (tt.name.IndexOf(model.value, StringComparison.OrdinalIgnoreCase) != -1) : false) || (model.filterDNS ? tt.dnsName.Contains(model.value) : false) || (model.filterService1 ? tt.serviceIP1.Contains(model.value) : false) || (model.filterService2 ? tt.serviceIP2.Contains(model.value) : false) || (model.filterVSIP ? tt.VSIP.Contains(model.value) : false) || (model.filterIntersiteIP ? tt.interSiteIP.Contains(model.value) : false) || (model.filterExternalIP ? tt.externalSRCIP.Contains(model.value) : false) || (model.filterDescription ? tt.description.Contains(model.value) : false)));
				List<IPAMOutput> list = source.Select((IPLists ip) => new IPAMOutput(ip)).ToList();
				if (list.Count > 0)
				{
					result.success = true;
				}
				result.rows = list;
				return new JsonResult(result);
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetSearchDevice():335");
		}
		return new JsonResult(result);
	}

	[HttpPost]
	[Authorize(Roles = "IPAMWrite")]
	public async Task<ResultOptions> EditIPAM([FromForm] IPAMInput model)
	{
		try
		{
			IPLists ipam = null;
			if (model.id == "new")
			{
				ipam = new IPLists();
			}
			else
			{
				ipam = GetLocalIPListByID(model.id);
			}
			if (string.IsNullOrEmpty(model.subnet))
			{
				await GetResultErrorAsync("nullSubnet", "اطلاعات Subnet تکمیل نشده است.");
			}
			else if (ipam == null)
			{
				await GetResultErrorAsync("nullSubnet", "اطلاعات اشتباه ارسال شده است!");
			}
			else if (IsLockIPAM(ipam))
			{
				await GetResultErrorAsync("useSubnet", "کاربر '" + ipam.lastUpdate.user.name + "' در حال ویرایش اطلاعات این سابنت می باشد");
			}
			else
			{
				IPLists byNetwork = GetByNetwork(model.subnet, model.networkName, model.siteMain);
				IPNetwork2 ip = IPNetwork2.Parse(model.subnet);
				Sites sites = null;
				List<Sites> list = _cachProvider._AllSites.Where((Sites tt) => tt.main == model.siteMain).ToList();
				if (list.Count > 0)
				{
					sites = list.FirstOrDefault((Sites tt) => tt.secondary == model.siteSecondary);
				}
				AccessNetwork accessNetwork;
				if (ip.ListIPAddress().Count > 600L)
				{
					await GetResultErrorAsync("errorSubnet", "آدرس Subnet به صورت صحیح تنظیم نشده است. و بیشتر از 599 آدرس IP نمی توانید در زیر مجموعه تعریف کنید");
				}
				else if (string.IsNullOrEmpty(model.subnet) || ip.ListIPAddress().Count < 3L)
				{
					await GetResultErrorAsync("errorSubnet", "آدرس Subnet به صورت صحیح تنظیم نشده است.");
				}
				else if ((byNetwork != null && byNetwork.id != ipam.id && !model.isCopy) || (model.isCopy && byNetwork != null))
				{
					await GetResultErrorAsync("existSubnet", "شبکه دیگری با همین سابنت و نام شبکه و سایت تنظیم شده است.");
				}
				else if (!string.IsNullOrEmpty(model.siteSecondary) && sites == null)
				{
					await GetResultErrorAsync("nullSite2", "نام سایت ثانویه اشتباه تنظیم شده است.");
				}
				else if ((accessNetwork = base._accessUser.ipams.FirstOrDefault((AccessNetwork tt) => tt.network == model.networkName)) == null)
				{
					await GetResultErrorAsync("notAccessIpamNetwork", "دسترسی برای تعریف/ویرایش Subnet در شبکه انتخاب شده را ندارید.");
				}
				else if (accessNetwork.access != "write")
				{
					await GetResultErrorAsync("notAccessIpamNetwork", "دسترسی برای تعریف/ویرایش Subnet در شبکه انتخاب شده را ندارید.");
				}
				else
				{
					ipam.lastUpdate = new LastUpdateData(base.CurrentUser.mainUserInfo, DateTime.Now, _system: false);
					ipam.siteMain = model.siteMain;
					ipam.siteSecondary = model.siteSecondary;
					ipam.networkName = model.networkName;
					ipam.vLanNumber = model.vLan;
					ipam.vLanName = model.vLanName;
					ipam.vrf = model.vrf;
					ipam.vni = model.vni;
					ipam.isSecret = model.isSecret;
					IPLists iPLists = ipam;
					iPLists.serviceName = await GetIPServiceAsync(model.service, model.networkName);
					ipam.description = model.description;
					ipam.tags = SetOrdinalIgnoreCase(model.tags, ipTags);
					if (string.IsNullOrEmpty(ipam.subNet))
					{
						ipam.network = ip.Network.ToString();
						ipam.netmask = ip.Netmask.ToString();
						IPAddressCollection iPAddressCollection = ip.ListIPAddress();
						foreach (IPAddress item in (IEnumerable<IPAddress>)iPAddressCollection)
						{
							IPListItem iPListItem = new IPListItem();
							iPListItem.ipAddress = item.ToString();
							iPListItem.isEdge = false;
							if (iPListItem.ipAddress == iPAddressCollection[0].ToString() || iPListItem.ipAddress == iPAddressCollection[iPAddressCollection.Count - 1].ToString())
							{
								iPListItem.isEdge = true;
							}
							ipam.devices.Add(iPListItem);
						}
						ipam.subNet = model.subnet;
					}
					if (!base._accessUser.ipams.Any((AccessNetwork tt) => tt.network == ipam.networkName && tt.access == "write"))
					{
						await GetResultErrorAsync("accessIpam", "عدم دسترسی برای شما جهت افزودن SubNet در این نام شبکه و سرویس!");
					}
					else
					{
						PublicJsonResult publicJsonResult;
						if (model.id == "new" || model.isCopy)
						{
							if (model.isCopy)
							{
								ipam.id = ObjectId.GenerateNewId();
							}
							publicJsonResult = await _ipListService.CreateAsync(ipam);
							_cachProvider.ClearCacheIPList();
						}
						else
						{
							publicJsonResult = await _ipListService.UpdateAsync(ipam);
						}
						if (publicJsonResult.Status == 1)
						{
							if (!(model.id == "new"))
							{
								await AddEventLogAsync("CMDB-Edit-IPAM", "Edit Subnet: " + ipam.subNet);
							}
							else
							{
								await AddEventLogAsync("CMDB-Add-IPAM", "Add Subnet: " + ipam.subNet);
							}
							resultOption.result = new IPAMOutput(ipam);
							await GetResultOkAsync("addSuccess", "اطلاعات با موفقیت در سامانه اضافه/ویرایش شد.");
						}
					}
				}
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "EditIPAM():239");
			resultOption.status = -1;
			resultOption.message = ex.Message;
		}
		return resultOption;
	}

	[HttpPost]
	[Authorize(Roles = "IPAMWrite")]
	public async Task<ResultOptions> DeleteIPAM([FromForm] IPAMInput model)
	{
		_ = 6;
		try
		{
			IPLists ipam = GetLocalIPListByID(model.id);
			AccessNetwork accessNetwork;
			if (ipam == null)
			{
				await GetResultErrorAsync("nullSubnet", "اطلاعات اشتباه ارسال شده است!");
			}
			else if (IsLockIPAM(ipam))
			{
				await GetResultErrorAsync("useSubnet", "کاربر '" + ipam.lastUpdate.user.name + "' در حال ویرایش اطلاعات این سابنت می باشد");
			}
			else if ((accessNetwork = base._accessUser.ipams.FirstOrDefault((AccessNetwork tt) => tt.network == ipam.networkName)) == null)
			{
				await GetResultErrorAsync("notAccessIpamNetwork", "دسترسی برای حذف/ویرایش Subnet در شبکه انتخاب شده را ندارید.");
			}
			else if (accessNetwork.access != "write")
			{
				await GetResultErrorAsync("notAccessIpamNetwork", "دسترسی برای حذف/ویرایش Subnet در شبکه انتخاب شده را ندارید.");
			}
			else
			{
				PublicJsonResult publicJsonResult = await _ipListService.DeleteAsync(ipam.id);
				_cachProvider.ClearCacheIPList();
				if (publicJsonResult.Status == 1)
				{
					await AddEventLogAsync("CMDB-Delete-IPAM", "Delete Subnet: " + ipam.subNet);
					await GetResultOkAsync("deleteSuccess", "اطلاعات سابنت در سامانه حذف شد.");
				}
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "DeleteIPAM():477");
			resultOption.status = -1;
			resultOption.message = ex.Message;
		}
		return resultOption;
	}

	[HttpPost]
	[Authorize(Roles = "IPAMWrite")]
	public async Task<ResultOptions> EditIPAMDetails([FromForm] IPAMDetailsInput model)
	{
		try
		{
			await AddEventLogAsync("CMDB-Edit-IPAM-Details", "Edit IPAddress: " + model.ipAddress);
			IPLists ipam = GetLocalIPListByID(model.parentId);
			if (ipam == null)
			{
				await GetResultErrorAsync("nullSubnet", "اطلاعات اشتباه ارسال شده است! لطفا صفحه را بروز رسانی فرمایید");
			}
			else if (IsLockIPAM(ipam))
			{
				await GetResultErrorAsync("useSubnet", "کاربر '" + ipam.lastUpdate.user.name + "' در حال ویرایش اطلاعات این سابنت می باشد");
			}
			else
			{
				IPListItem ipItem = ipam.devices.Where((IPListItem tt) => tt.id == ObjectId.Parse(model.id) && tt.ipAddress == model.ipAddress).FirstOrDefault();
				if (ipItem == null)
				{
					await GetResultErrorAsync("nullSubnet", "اطلاعات اشتباه ارسال شده است! لطفا صفحه را بروز رسانی فرمایید");
				}
				else if (!base._accessUser.ipams.Any((AccessNetwork tt) => tt.network == ipam.networkName && tt.access == "write"))
				{
					await GetResultErrorAsync("accessIpam", "عدم دسترسی برای شما جهت افزودن/ویرایش SubNet در این شبکه !");
				}
				else if (string.IsNullOrEmpty(model.name) && model.status != "UnUsed")
				{
					await GetResultErrorAsync("nullSubnetName", "نام تجهیز وارد نشده! لطف دوباره همین سطر را ویرایش کنید");
				}
				else
				{
					ipItem.name = model.name;
					ipItem.status = model.status;
					ipItem.dnsName = model.dnsName;
					ipItem.description = model.description;
					ipItem.type = model.type;
					ipItem.interSiteIP = model.interSiteIP;
					ipItem.serviceIP1 = model.serviceIP1;
					ipItem.serviceIP2 = model.serviceIP2;
					ipItem.VSIP = model.vsIP;
					ipItem.externalSRCIP = model.externalSRCIP;
					ipam.lastUpdate = new LastUpdateData(base.CurrentUser.mainUserInfo, DateTime.Now, _system: true);
					await _ipListService.UpdateAsync(ipam);
					resultOption.htmlString1 = ipam.devices.Where((IPListItem tt) => tt.status == "Used").ToList().Count.ToString();
					resultOption.htmlString2 = (ipam.devices.Where((IPListItem tt) => tt.status == "UnUsed").ToList().Count - 2).ToString();
					await GetResultOkAsync("addSuccess", "اطلاعات با موفقیت به سامانه اضافه گردید.");
				}
				if (ipItem != null)
				{
					resultOption.result = new JsonResult(new IPAMDetailsInput
					{
						id = ipItem.id.ToString(),
						parentId = ipam.id.ToString(),
						name = ipItem.name,
						ipAddress = ipItem.ipAddress,
						type = ipItem.type,
						dnsName = ipItem.dnsName,
						status = ipItem.status,
						serviceIP1 = ipItem.serviceIP1,
						serviceIP2 = ipItem.serviceIP2,
						vsIP = ipItem.VSIP,
						interSiteIP = ipItem.interSiteIP,
						externalSRCIP = ipItem.externalSRCIP,
						description = ipItem.description
					});
				}
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "EditIPAMDetails():310");
			resultOption.status = -1;
			resultOption.message = "لطف دوباره همین سطر را ویرایش کنید ! " + ex.Message;
		}
		return resultOption;
	}

	[HttpPost]
	[Authorize(Roles = "IPAMWrite")]
	public async Task<ResultOptions> EditIPServices([FromForm] IPServiceInput model)
	{
		try
		{
			IPServices service = ((!(model.id == "new")) ? GetLocalIPServiceByID(model.id) : new IPServices());
			if (string.IsNullOrEmpty(model.nameEn))
			{
				await GetResultErrorAsync("nullServiceName", "اطلاعات نام سرویس تکمیل نشده است.");
			}
			else if (service == null)
			{
				await GetResultErrorAsync("nullService", "اطلاعات اشتباه ارسال شده است!");
			}
			else
			{
				IPServices serviceByNetwork = GetServiceByNetwork(model.network, model.nameEn);
				AccessNetwork accessNetwork;
				if (serviceByNetwork != null && serviceByNetwork.id != service.id)
				{
					await GetResultErrorAsync("existService", "سرویس دیگری با همین نام و نام شبکه تنظیم شده است.");
				}
				else if ((accessNetwork = base._accessUser.ipams.FirstOrDefault((AccessNetwork tt) => tt.network == model.network)) == null)
				{
					await GetResultErrorAsync("notAccessIPServiceNetwork", "دسترسی برای تعریف/ویرایش سرویس در شبکه انتخاب شده را ندارید.");
				}
				else if (accessNetwork.access != "write")
				{
					await GetResultErrorAsync("notAccessIPServiceNetwork", "دسترسی برای تعریف/ویرایش سرویس در شبکه انتخاب شده را ندارید.");
				}
				else
				{
					service.nameEn = model.nameEn;
					service.nameFa = model.nameFa;
					service.network = model.network;
					service.info = model.info;
					PublicJsonResult publicJsonResult;
					if (model.id == "new")
					{
						publicJsonResult = await _ipServiceService.CreateAsync(service);
						_cachProvider.ClearCacheIPServices();
					}
					else
					{
						publicJsonResult = await _ipServiceService.UpdateAsync(service);
					}
					if (publicJsonResult.Status == 1)
					{
						if (!(model.id == "new"))
						{
							await AddEventLogAsync("CMDB-Edit-IPAM-Service", "Edit IPAM Service: " + service.nameEn);
						}
						else
						{
							await AddEventLogAsync("CMDB-Add-IPAM-Service", "Add IPAM Service: " + service.nameEn);
						}
						resultOption.result = new IPServiceInput(service);
						await GetResultOkAsync("addSuccess", "اطلاعات با موفقیت در سامانه اضافه/ویرایش شد.");
					}
				}
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "EditIPServices():588");
			resultOption.status = -1;
			resultOption.message = ex.Message;
		}
		return resultOption;
	}

	[HttpPost]
	[Authorize(Roles = "IPAMWrite")]
	public async Task<JsonResult> UploadIPListFile(UploadCMDBFile cmdbFile)
	{
		_ = 3;
		try
		{
			ExcelPackage.LicenseContext = LicenseContext.NonCommercial;
			ExcelPackage excelPackage = new ExcelPackage(cmdbFile.excelFile.OpenReadStream());
			ExcelWorksheets worksheets = excelPackage.Workbook.Worksheets;
			resultOption.message = string.Empty;
			bool flag = true;
			if (HasCorrectUploadIpamTemplate(excelPackage))
			{
				List<IPLists> ipSubnets = null;
				ExcelWorksheet excelWorksheet = worksheets.Where((ExcelWorksheet tt) => string.Equals(tt.Name, "Subnet", StringComparison.OrdinalIgnoreCase)).SingleOrDefault();
				if (excelWorksheet != null)
				{
					resultOption.message = FillTempSubnet(excelWorksheet, out ipSubnets);
				}
				else
				{
					flag = false;
				}
				if (string.IsNullOrEmpty(resultOption.message))
				{
					excelWorksheet = worksheets.Where((ExcelWorksheet tt) => string.Equals(tt.Name, "IP Details", StringComparison.OrdinalIgnoreCase)).SingleOrDefault();
					if (excelWorksheet != null)
					{
						resultOption.message = UpdateTempIP(excelWorksheet, ipSubnets);
					}
					else
					{
						flag = false;
					}
				}
				if (!flag)
				{
					resultOption.message = "نام کاربرگ ها در فایل اکسل مطابق قالب تعریف شده نمی باشد " + Environment.NewLine + resultOption.message;
				}
				else if (string.IsNullOrEmpty(resultOption.message))
				{
					foreach (IPLists ipSubnetTemp in ipSubnets)
					{
						IPLists iPLists = _cachProvider._AllIPList.Where((IPLists tt) => tt.networkName == ipSubnetTemp.networkName && tt.subNet == ipSubnetTemp.subNet && tt.serviceName == ipSubnetTemp.serviceName).FirstOrDefault();
						if (iPLists == null)
						{
							await _ipListService.CreateAsync(ipSubnetTemp);
							continue;
						}
						ipSubnetTemp.id = iPLists.id;
						foreach (IPListItem ipamServer in iPLists.devices)
						{
							IPListItem iPListItem = ipSubnetTemp.devices.Where((IPListItem tt) => tt.ipAddress == ipamServer.ipAddress).FirstOrDefault();
							if (iPListItem == null)
							{
								continue;
							}
							if (iPListItem.status == "Used")
							{
								iPListItem.id = ipamServer.id;
								if (string.IsNullOrEmpty(iPListItem.name))
								{
									iPListItem.name = ipamServer.name;
								}
								if (string.IsNullOrEmpty(iPListItem.dnsName))
								{
									iPListItem.dnsName = ipamServer.dnsName;
								}
								if (string.IsNullOrEmpty(iPListItem.type))
								{
									iPListItem.type = ipamServer.type;
								}
								if (string.IsNullOrEmpty(iPListItem.serviceIP1))
								{
									iPListItem.serviceIP1 = ipamServer.serviceIP1;
								}
								if (string.IsNullOrEmpty(iPListItem.serviceIP2))
								{
									iPListItem.serviceIP2 = ipamServer.serviceIP2;
								}
								if (string.IsNullOrEmpty(iPListItem.VSIP))
								{
									iPListItem.VSIP = ipamServer.VSIP;
								}
								if (string.IsNullOrEmpty(iPListItem.interSiteIP))
								{
									iPListItem.interSiteIP = ipamServer.interSiteIP;
								}
								if (string.IsNullOrEmpty(iPListItem.externalSRCIP))
								{
									iPListItem.externalSRCIP = ipamServer.externalSRCIP;
								}
								if (string.IsNullOrEmpty(iPListItem.description))
								{
									iPListItem.description = ipamServer.description;
								}
							}
							else
							{
								iPListItem.id = ipamServer.id;
								iPListItem.status = ipamServer.status;
								iPListItem.name = ipamServer.name;
								iPListItem.dnsName = ipamServer.dnsName;
								iPListItem.type = ipamServer.type;
								iPListItem.serviceIP1 = ipamServer.serviceIP1;
								iPListItem.serviceIP2 = ipamServer.serviceIP2;
								iPListItem.VSIP = ipamServer.VSIP;
								iPListItem.interSiteIP = ipamServer.interSiteIP;
								iPListItem.externalSRCIP = ipamServer.externalSRCIP;
								iPListItem.description = ipamServer.description;
							}
						}
						await _ipListService.UpdateAsync(ipSubnetTemp);
					}
					string text = string.Join(",", ipSubnets.Select((IPLists tt) => tt.subNet).ToList());
					_cachProvider.ClearCacheIPList();
					await AddEventLogAsync("CMDB-Bulk-Upload-IPAM", "Upload Subnet: " + text);
					await GetResultOkAsync("cmdb.updateOk", "اطلاعات تجهیزات فایل بارگزاری شده با موفقیت در بانک اطلاعاتی اضافه شد");
				}
			}
		}
		catch (Exception ex)
		{
			ResultOptions resultOptions = resultOption;
			resultOptions.message = resultOptions.message + "<br/>" + ex.Message;
		}
		return Json(resultOption);
	}

	internal IPLists GetLocalIPListByID(string id)
	{
		return MyIPAM.FirstOrDefault((IPLists tt) => tt.id == ObjectId.Parse(id));
	}

	internal IPServices GetLocalIPServiceByID(string id)
	{
		return MyIPServices.FirstOrDefault((IPServices tt) => tt.id == ObjectId.Parse(id));
	}

	internal IPLists GetByNetwork(string subNet, string network, string siteName)
	{
		return _cachProvider._AllIPList.FirstOrDefault((IPLists p) => p.subNet == subNet && string.Equals(p.networkName, network, StringComparison.OrdinalIgnoreCase) && string.Equals(p.siteMain, siteName, StringComparison.OrdinalIgnoreCase));
	}

	internal IPServices GetServiceByNetwork(string network, string service)
	{
		return _cachProvider._AllIPServices.FirstOrDefault((IPServices p) => string.Equals(p.network, network, StringComparison.OrdinalIgnoreCase) && string.Equals(p.nameEn, service, StringComparison.OrdinalIgnoreCase));
	}

	internal bool IsLockIPAM(IPLists ipam)
	{
		if (DateTime.Now.AddMinutes(-10.0) < ipam.lastUpdate.date)
		{
			return ipam.lastUpdate.user.UserName != base.CurrentUser.mainUserInfo.UserName;
		}
		return false;
	}

	internal List<string> GetAccessEditIpamNetworkList()
	{
		List<string> list = (from tt in base._accessUser.ipams
			where tt.access == "write"
			select tt.network).ToList();
		if (list.Count <= 0)
		{
			return new List<string>();
		}
		return list;
	}

	internal async Task<string> GetIPServiceAsync(string serviceName, string networkName)
	{
		List<string> accessNetworks = GetAccessEditIpamNetworkList();
		string text = (from tt in MyIPServices
			where accessNetworks.Any((string a) => a == tt.network) && string.Equals(tt.network, networkName, StringComparison.OrdinalIgnoreCase) && string.Equals(tt.nameEn, serviceName, StringComparison.OrdinalIgnoreCase)
			select tt.nameEn).FirstOrDefault();
		if (string.IsNullOrEmpty(text) && !string.IsNullOrEmpty(serviceName))
		{
			IPServices iPServices = new IPServices();
			iPServices.nameEn = serviceName;
			iPServices.network = networkName;
			await _ipServiceService.CreateAsync(iPServices);
			_cachProvider.ClearCacheIPServices();
			return serviceName;
		}
		return (text == null) ? string.Empty : text;
	}

	internal string SetOrdinalIgnoreCase(string items, List<string> compareList)
	{
		string[] allTag = items.Replace("\"", "").Replace("[", "").Replace("]", "")
			.Split(",");
		int i;
		for (i = 0; i < allTag.Length; i++)
		{
			string text = compareList.Where((string tt) => tt.Equals(allTag[i], StringComparison.OrdinalIgnoreCase)).FirstOrDefault();
			if (!string.IsNullOrEmpty(text))
			{
				allTag[i] = text;
			}
		}
		return string.Join(",", allTag);
	}

	internal bool HasCorrectUploadIpamTemplate(ExcelPackage package)
	{
		bool flag = false;
		try
		{
			ColumnList = new Dictionary<string, int>();
			ColumnList.Add("Service", 1);
			ColumnList.Add("Subnet", 2);
			ColumnList.Add("Network", 3);
			ColumnList.Add("Site", 4);
			ColumnList.Add("SubSite", 5);
			ColumnList.Add("Tag", 6);
			ColumnList.Add("VLAN", 7);
			ColumnList.Add("VLAN Name", 8);
			ColumnList.Add("VRF", 9);
			ColumnList.Add("VNI", 10);
			ColumnList.Add("Gateway", 11);
			ColumnList.Add("IP Address", 3);
			ColumnList.Add("HostName", 4);
			ColumnList.Add("DNS Name", 5);
			ColumnList.Add("Type", 6);
			ColumnList.Add("Service IP 1", 7);
			ColumnList.Add("Service IP 2", 8);
			ColumnList.Add("VS IP", 9);
			ColumnList.Add("Intersite IP", 10);
			ColumnList.Add("External SRC IP", 11);
			ColumnList.Add("Description", 12);
			ExcelWorksheets worksheets = package.Workbook.Worksheets;
			if (worksheets.Count >= 2)
			{
				if (worksheets[0].Name.Equals("Subnet", StringComparison.OrdinalIgnoreCase))
				{
					ExcelWorksheet item = worksheets[0];
					if (GetStringOfValue(item, 1, "Network").Equals("Network", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item, 1, "Subnet").Equals("Subnet", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item, 1, "Service").Equals("Service", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item, 1, "Site").Equals("Site", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item, 1, "SubSite").Equals("SubSite", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item, 1, "Tag").Equals("Tag", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item, 1, "VLAN").Equals("VLAN", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item, 1, "VLAN Name").Equals("VLAN Name", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item, 1, "VRF").Equals("VRF", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item, 1, "VNI").Equals("VNI", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item, 1, "Gateway").Equals("Gateway", StringComparison.OrdinalIgnoreCase))
					{
						flag = true;
					}
				}
				if (flag && worksheets[1].Name.Equals("IP Details", StringComparison.OrdinalIgnoreCase))
				{
					ExcelWorksheet item2 = worksheets[1];
					flag = ((GetStringOfValue(item2, 1, "Service").Equals("Service", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "Subnet").Equals("Subnet", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "IP Address").Equals("IP Address", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "HostName").Equals("HostName", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "DNS Name").Equals("DNS Name", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "Type").Equals("Type", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "Service IP 1").Equals("Service IP 1", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "Service IP 2").Equals("Service IP 2", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "VS IP").Equals("VS IP", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "Intersite IP").Equals("Intersite IP", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "External SRC IP").Equals("External SRC IP", StringComparison.OrdinalIgnoreCase) && GetStringOfValue(item2, 1, "Description").Equals("Description", StringComparison.OrdinalIgnoreCase)) ? true : false);
				}
			}
		}
		catch (Exception)
		{
			flag = false;
		}
		return flag;
	}

	internal string FillTempSubnet(ExcelWorksheet worksheet, out List<IPLists> ipSubnets)
	{
		ipSubnets = new List<IPLists>();
		string text = string.Empty;
		try
		{
			AccessNetwork accessNetwork = null;
			int column = worksheet.Dimension.End.Column;
			int row = worksheet.Dimension.End.Row;
			IPLists ipSubnet = null;
			for (int i = 2; i <= row; i++)
			{
				ipSubnet = new IPLists();
				ipSubnet.networkName = GetStringOfValue(worksheet, i, "Network");
				if ((accessNetwork = base._accessUser.ipams.FirstOrDefault((AccessNetwork tt) => tt.network == ipSubnet.networkName)) == null)
				{
					text += $"دسترسی برای تعریف/ویرایش Subnet در شبکه {ipSubnet.networkName} که در سطر {i} تعریف شده را ندارید {Environment.NewLine}";
				}
				else if (accessNetwork.access != "write")
				{
					text += $"دسترسی برای تعریف/ویرایش Subnet در شبکه {ipSubnet.networkName} که در سطر {i} تعریف شده را ندارید {Environment.NewLine}";
				}
				ipSubnet.subNet = GetStringOfValue(worksheet, i, "Subnet").Replace(" ", string.Empty);
				ipSubnet.tags = GetStringOfValue(worksheet, i, "Tag").Replace(" ,", ",").Replace(", ", ",").Replace(",", ";");
				ipSubnet.serviceName = GetStringOfValue(worksheet, i, "Service");
				ipSubnet.siteMain = GetStringOfValue(worksheet, i, "Site");
				ipSubnet.siteSecondary = GetStringOfValue(worksheet, i, "SubSite");
				ipSubnet.vLanNumber = GetStringOfValue(worksheet, i, "VLAN");
				ipSubnet.vLanName = GetStringOfValue(worksheet, i, "VLAN Name");
				ipSubnet.vrf = GetStringOfValue(worksheet, i, "VRF");
				ipSubnet.vni = GetStringOfValue(worksheet, i, "VNI");
				ipSubnet.description = GetStringOfValue(worksheet, i, "Gateway");
				ipSubnet.isSecret = true;
				ipSubnet.lastUpdate = new LastUpdateData(base.CurrentUser.userInfo, DateTime.Now, _system: false);
				bool flag = true;
				ipSubnet.devices = new List<IPListItem>();
				if (string.IsNullOrEmpty(ipSubnet.subNet))
				{
					flag = false;
					text += $"ستون سابنت در سطر {i} خالی تنظیم شده {Environment.NewLine}";
				}
				else
				{
					IPNetwork2 iPNetwork = IPNetwork2.Parse(ipSubnet.subNet);
					ipSubnet.network = iPNetwork.Network.ToString();
					ipSubnet.netmask = iPNetwork.Netmask.ToString();
					IPAddressCollection iPAddressCollection = iPNetwork.ListIPAddress();
					if (iPAddressCollection.Count > 600L)
					{
						flag = false;
						text += $"سابنت در سطر {i} اشتباه تنظیم شده و بیشتر از 600 آدرس IP در زیر مجموعه دارد که قابل ثبت نیست {Environment.NewLine}";
					}
					else
					{
						foreach (IPAddress item in (IEnumerable<IPAddress>)iPAddressCollection)
						{
							IPListItem iPListItem = new IPListItem();
							iPListItem.ipAddress = item.ToString();
							iPListItem.isEdge = false;
							if (iPListItem.ipAddress == iPAddressCollection[0].ToString() || iPListItem.ipAddress == iPAddressCollection[iPAddressCollection.Count - 1].ToString())
							{
								iPListItem.isEdge = true;
							}
							if (iPListItem.ipAddress == ipSubnet.description)
							{
								iPListItem.description = "Gateway";
							}
							ipSubnet.devices.Add(iPListItem);
						}
					}
				}
				if (flag)
				{
					ipSubnets.Add(ipSubnet);
				}
			}
		}
		catch (Exception ex)
		{
			text = text + "<br/>IPAM: " + ex.Message;
		}
		return text;
	}

	internal string UpdateTempIP(ExcelWorksheet worksheet, List<IPLists> ipSubnets)
	{
		string text = string.Empty;
		try
		{
			int column = worksheet.Dimension.End.Column;
			int row = worksheet.Dimension.End.Row;
			string serviceName = string.Empty;
			string subnet = string.Empty;
			string ipAddress = string.Empty;
			for (int i = 2; i <= row; i++)
			{
				serviceName = GetStringOfValue(worksheet, i, "Service");
				subnet = GetStringOfValue(worksheet, i, "Subnet").Replace(" ", string.Empty);
				ipAddress = GetStringOfValue(worksheet, i, "IP Address").Replace(" ", string.Empty);
				IPLists iPLists = ipSubnets.Where((IPLists tt) => tt.serviceName == serviceName && tt.subNet == subnet).FirstOrDefault();
				if (iPLists == null)
				{
					text += $"نام سرویس با سابنت ثبت شده در سطر {i} در کاربرگ آدرس IP ها در لیست اطلاعات داده شده در کاربرگ اول ثبت نشده است {Environment.NewLine}";
					continue;
				}
				IPListItem iPListItem = iPLists.devices.Where((IPListItem a) => a.ipAddress == ipAddress).FirstOrDefault();
				if (iPListItem == null || iPListItem.isEdge)
				{
					text += $"آدرس IP در سطر {i} خالی تنظیم شده یا اشتباه است یا لبه می باشد و نباید نوشته شود {Environment.NewLine}";
					continue;
				}
				iPListItem.status = "Used";
				iPListItem.name = GetStringOfValue(worksheet, i, "HostName");
				iPListItem.dnsName = GetStringOfValue(worksheet, i, "DNS Name");
				iPListItem.type = GetStringOfValue(worksheet, i, "Type");
				iPListItem.serviceIP1 = GetStringOfValue(worksheet, i, "Service IP 1");
				iPListItem.serviceIP2 = GetStringOfValue(worksheet, i, "Service IP 2");
				iPListItem.VSIP = GetStringOfValue(worksheet, i, "VS IP");
				iPListItem.interSiteIP = GetStringOfValue(worksheet, i, "Intersite IP");
				iPListItem.externalSRCIP = GetStringOfValue(worksheet, i, "External SRC IP");
				iPListItem.description = GetStringOfValue(worksheet, i, "Description");
			}
		}
		catch (Exception ex)
		{
			text = text + "<br/>IPAM: " + ex.Message;
		}
		return text;
	}

	internal string GetStringOfValue(ExcelWorksheet item, int rowNumber, string value)
	{
		string text = string.Empty;
		if (item.Cells[rowNumber, ColumnList[value]].Value != null)
		{
			text = item.Cells[rowNumber, ColumnList[value]].Value.ToString().Trim().ImproveText();
		}
		if (text.Replace("<context>", string.Empty).HasScriptText())
		{
			text = ((!text.Replace("<", "*").Replace(">", "*").HasScriptText()) ? text.Replace("<", "*").Replace(">", "*") : "*****");
		}
		return (text.ToLower() == "unknown") ? string.Empty : text;
	}

	[BreadCrumb(Title = "Networks", Order = 2, GlyphIcon = "icon icon-monitor-share ")]
	[AddHeader("Networks")]
	[Authorize(Roles = "CMDB")]
	public async Task<IActionResult> Networks()
	{
		bool isAll = base._accessUser.networks.Any((AccessNetwork a) => a.network == "All");
		base.ViewData["networks"] = (from tt in _appSettings.Value.NetworkTags.Split(",").ToList()
			where tt.ToLower() != "all" && tt.ToLower() != "other" && (isAll || base._accessUser.networks.Any((AccessNetwork a) => a.network == tt))
			select tt).ToList();
		base.ViewData["sites"] = new List<string>();
		if (!isAll && base._accessUser.networks.Count == 1)
		{
			base.ViewData["sites"] = (from tt in _cachProvider._AllDevices
				where string.Equals(tt.tag.network, base._accessUser.networks[0].network, StringComparison.OrdinalIgnoreCase)
				select tt.site).Distinct().ToList();
		}
		await AddEventLogAsync("CMDB-Show-Networks-Page", base.CurrentUser.mainUserInfo.UserName);
		return View();
	}

	[HttpPost]
	[AjaxOnly]
	public async Task<JsonResult> GetSitesByNetwork([FromForm] GridParam gridParam)
	{
		try
		{
			List<DocumentList> list = new List<DocumentList>();
			List<DocumentList> list2 = new List<DocumentList>();
			if (string.IsNullOrEmpty(gridParam.id))
			{
				await GetResultErrorAsync("cmdb.nullNetwork", "نام شبکه برای دریافت لیست سایت صحیح وارد نشده است!");
			}
			else
			{
				string[] array = gridParam.id.Split(',');
				string[] array2 = array;
				foreach (string item in array2)
				{
					if (string.IsNullOrWhiteSpace(item))
					{
						continue;
					}
					DocumentList item2 = new DocumentList
					{
						Header = item,
						NameList = (from tt in _cachProvider._AllDevices
							where string.Equals(tt.tag.network, item, StringComparison.OrdinalIgnoreCase)
							select tt.site).Distinct().ToList()
					};
					list.Add(item2);
					List<string> list3 = new List<string>();
					List<List<string>> list4 = (from tt in _cachProvider._AllDevices
						where string.Equals(tt.tag.network, item, StringComparison.OrdinalIgnoreCase)
						select tt.tag.tags).ToList();
					foreach (List<string> item4 in list4)
					{
						foreach (string item5 in item4)
						{
							if (!list3.Contains(item5))
							{
								list3.Add(item5);
							}
						}
					}
					if (list3.Count > 0)
					{
						list3.Insert(0, "");
					}
					DocumentList item3 = new DocumentList
					{
						Header = item,
						NameList = list3
					};
					list2.Add(item3);
				}
				resultOption.status = 1;
				resultOption.result = new
				{
					sites = list,
					tags = list2
				};
			}
		}
		catch (Exception ex)
		{
			resultOption.message = ex.Message;
			_logger.LogCritical(ex, "GetSitesByNetwork():972");
		}
		return Json(resultOption);
	}

	[HttpPost]
	[AjaxOnly]
	public async Task<JsonResult> GetGraphNetwork([FromForm] GraphNetworkView gridNetwork)
	{
		try
		{
			await AddEventLogAsync("CMDB-Show-Network", gridNetwork.network);
			int num = 0;
			List<string> list = new List<string>();
			if (!string.IsNullOrEmpty(gridNetwork.sites))
			{
				list = gridNetwork.sites.Split(',').ToList();
			}
			bool isAll = base._accessUser.networks.Any((AccessNetwork a) => a.network == "All");
			List<AccessNetwork> list2 = (from tt in base._accessUser.networks
				where (isAll && tt.network.Length > 0) || tt.network.Any((char a) => tt.network.ToUpper().ToString().Contains(gridNetwork.network.ToUpper()))
				select (tt)).ToList();
			if (list2.Count == 0 || string.IsNullOrEmpty(gridNetwork.network))
			{
				await GetResultErrorAsync("cmdb.notAccessNet", "دسترسی به شبکه انتخاب شده برای شما تعریف نشده است!");
			}
			else
			{
				IMongoCollection<Links> collection = _mongoDbContext.GetCollection<Links>("Links");
				new List<Links>();
				List<Links> list3 = (from tt in collection.AsQueryable()
					where tt.networks.Any((string n) => string.Equals(n, gridNetwork.network, StringComparison.OrdinalIgnoreCase))
					select (tt)).ToList();
				List<DeviceForNetworkGraph> nodes = new List<DeviceForNetworkGraph>();
				List<LinkForNetworkGraph> list4 = new List<LinkForNetworkGraph>();
				List<string> SiteForLinks = new List<string>();
				foreach (Links item2 in list3)
				{
					Devint devint = new Devint();
					Devint devint2 = new Devint();
					bool flag = false;
					Devices deviceSide = GetDeviceSide(item2.side1, out devint);
					DeviceForNetworkGraph deviceForNetworkGraph = SetDeviceNode(deviceSide, item2.customerServices, ref nodes, gridNetwork.network, list, gridNetwork.deviceTag);
					Devices deviceSide2 = GetDeviceSide(item2.side2, out devint2);
					DeviceForNetworkGraph deviceForNetworkGraph2 = SetDeviceNode(deviceSide2, item2.customerServices, ref nodes, gridNetwork.network, list, gridNetwork.deviceTag);
					LinkForNetworkGraph linkForNetworkGraph = new LinkForNetworkGraph();
					if (deviceForNetworkGraph == null || deviceForNetworkGraph2 == null || deviceSide == null || deviceSide2 == null)
					{
						continue;
					}
					if (!gridNetwork.isSiteSeperation)
					{
						string group = (deviceForNetworkGraph2.group = string.Empty);
						deviceForNetworkGraph.group = group;
					}
					if (!(deviceForNetworkGraph.category == "SiteContainer") || !(deviceForNetworkGraph2.category == "SiteContainer") || (deviceForNetworkGraph.category == "cloud" && deviceForNetworkGraph2.category == "cloud"))
					{
						if (!SiteForLinks.Contains(deviceForNetworkGraph.site))
						{
							SiteForLinks.Add(deviceForNetworkGraph.site);
						}
						if (!SiteForLinks.Contains(deviceForNetworkGraph2.site))
						{
							SiteForLinks.Add(deviceForNetworkGraph2.site);
						}
					}
					if (deviceForNetworkGraph.category == "cloud" && deviceForNetworkGraph2.category != "cloud")
					{
						linkForNetworkGraph.from = deviceForNetworkGraph2.key;
						linkForNetworkGraph.to = deviceForNetworkGraph.key;
						linkForNetworkGraph.sourcePort = $"{devint2.type} {devint2.number}";
						linkForNetworkGraph.targetPort = $"{devint.type} {devint.number}";
						linkForNetworkGraph.targetDevice = deviceSide.name;
						linkForNetworkGraph.category = "linkToCloud";
						linkForNetworkGraph.linkType = item2.linkType;
						linkForNetworkGraph.site = deviceForNetworkGraph.site;
						linkForNetworkGraph.services = GetServices(item2.customerServices);
						if (num < deviceForNetworkGraph2.layer)
						{
							num = deviceForNetworkGraph2.layer;
						}
					}
					else if (deviceForNetworkGraph2.category == "cloud" && deviceForNetworkGraph.category != "cloud")
					{
						linkForNetworkGraph.from = deviceForNetworkGraph.key;
						linkForNetworkGraph.to = deviceForNetworkGraph2.key;
						linkForNetworkGraph.sourcePort = $"{devint.type} {devint.number}";
						linkForNetworkGraph.targetPort = $"{devint2.type} {devint2.number}";
						linkForNetworkGraph.targetDevice = deviceSide2.name;
						linkForNetworkGraph.category = "linkToCloud";
						linkForNetworkGraph.linkType = item2.linkType;
						linkForNetworkGraph.site = deviceForNetworkGraph.site;
						linkForNetworkGraph.services = GetServices(item2.customerServices);
						if (num < deviceForNetworkGraph.layer)
						{
							num = deviceForNetworkGraph.layer;
						}
					}
					else if (deviceForNetworkGraph.category == "cloud" && deviceForNetworkGraph2.category == "cloud")
					{
						linkForNetworkGraph.from = deviceForNetworkGraph.key;
						linkForNetworkGraph.to = deviceForNetworkGraph2.key;
						linkForNetworkGraph.sourcePort = $"{devint.type} {devint.number}";
						linkForNetworkGraph.targetPort = $"{devint2.type} {devint2.number}";
						linkForNetworkGraph.targetDevice = deviceSide2.name;
						linkForNetworkGraph.category = "linkToCloud";
						linkForNetworkGraph.linkType = item2.linkType;
						linkForNetworkGraph.site = deviceForNetworkGraph.site;
						linkForNetworkGraph.services = GetServices(item2.customerServices);
						if (num < deviceForNetworkGraph.layer)
						{
							num = deviceForNetworkGraph.layer;
						}
						flag = true;
					}
					else if (list.Count > 0 || !string.IsNullOrEmpty(gridNetwork.deviceTag))
					{
						if (deviceForNetworkGraph.category == "SiteContainer" && deviceForNetworkGraph2.category != "SiteContainer")
						{
							linkForNetworkGraph.from = deviceForNetworkGraph2.key;
							linkForNetworkGraph.to = deviceForNetworkGraph.key;
							linkForNetworkGraph.sourcePort = $"{devint2.type} {devint2.number}";
							linkForNetworkGraph.targetPort = $"{devint.type} {devint.number}";
							linkForNetworkGraph.targetDevice = deviceSide.name;
							linkForNetworkGraph.category = "linkToSite";
							linkForNetworkGraph.linkType = item2.linkType;
							linkForNetworkGraph.site = deviceForNetworkGraph.site;
							linkForNetworkGraph.services = GetServices(item2.customerServices);
							if (num < deviceForNetworkGraph2.layer)
							{
								num = deviceForNetworkGraph2.layer;
							}
							if (num < deviceForNetworkGraph.layer)
							{
								num = deviceForNetworkGraph.layer;
							}
						}
						else if (deviceForNetworkGraph2.category == "SiteContainer" && deviceForNetworkGraph.category != "SiteContainer")
						{
							linkForNetworkGraph.from = deviceForNetworkGraph.key;
							linkForNetworkGraph.to = deviceForNetworkGraph2.key;
							linkForNetworkGraph.sourcePort = $"{devint.type} {devint.number}";
							linkForNetworkGraph.targetPort = $"{devint2.type} {devint2.number}";
							linkForNetworkGraph.targetDevice = deviceSide2.name;
							linkForNetworkGraph.category = "linkToSite";
							linkForNetworkGraph.linkType = item2.linkType;
							linkForNetworkGraph.site = deviceForNetworkGraph.site;
							linkForNetworkGraph.services = GetServices(item2.customerServices);
							if (num < deviceForNetworkGraph.layer)
							{
								num = deviceForNetworkGraph.layer;
							}
							if (num < deviceForNetworkGraph2.layer)
							{
								num = deviceForNetworkGraph2.layer;
							}
						}
						else if (deviceForNetworkGraph.category == "SiteContainer" && deviceForNetworkGraph2.category == "SiteContainer")
						{
							flag = true;
						}
						else if (deviceForNetworkGraph.category != "SiteContainer" && deviceForNetworkGraph2.category != "SiteContainer")
						{
							linkForNetworkGraph.from = ((deviceSide.tag.level < deviceSide2.tag.level) ? deviceForNetworkGraph.key : deviceForNetworkGraph2.key);
							linkForNetworkGraph.to = ((deviceSide.tag.level < deviceSide2.tag.level) ? deviceForNetworkGraph2.key : deviceForNetworkGraph.key);
							linkForNetworkGraph.sourcePort = ((deviceSide.tag.level < deviceSide2.tag.level) ? $"{devint.type} {devint.number}" : $"{devint2.type} {devint2.number}");
							linkForNetworkGraph.targetPort = ((deviceSide.tag.level < deviceSide2.tag.level) ? $"{devint2.type} {devint2.number}" : $"{devint.type} {devint.number}");
							linkForNetworkGraph.targetDevice = linkForNetworkGraph.to;
							linkForNetworkGraph.category = string.Empty;
							linkForNetworkGraph.linkType = item2.linkType;
							linkForNetworkGraph.site = deviceForNetworkGraph.site;
							linkForNetworkGraph.services = GetServices(item2.customerServices);
							if (deviceSide.tag.level == deviceSide2.tag.level)
							{
								linkForNetworkGraph.category = "sameLayer";
							}
							if (num < deviceForNetworkGraph.layer)
							{
								num = deviceForNetworkGraph.layer;
							}
							if (num < deviceForNetworkGraph2.layer)
							{
								num = deviceForNetworkGraph2.layer;
							}
						}
					}
					else
					{
						linkForNetworkGraph.from = ((deviceSide.tag.level < deviceSide2.tag.level) ? deviceForNetworkGraph.key : deviceForNetworkGraph2.key);
						linkForNetworkGraph.to = ((deviceSide.tag.level < deviceSide2.tag.level) ? deviceForNetworkGraph2.key : deviceForNetworkGraph.key);
						linkForNetworkGraph.sourcePort = ((deviceSide.tag.level < deviceSide2.tag.level) ? $"{devint.type} {devint.number}" : $"{devint2.type} {devint2.number}");
						linkForNetworkGraph.targetPort = ((deviceSide.tag.level < deviceSide2.tag.level) ? $"{devint2.type} {devint2.number}" : $"{devint.type} {devint.number}");
						linkForNetworkGraph.targetDevice = linkForNetworkGraph.to;
						linkForNetworkGraph.category = string.Empty;
						linkForNetworkGraph.linkType = item2.linkType;
						linkForNetworkGraph.site = deviceForNetworkGraph.site;
						linkForNetworkGraph.services = GetServices(item2.customerServices);
						if (deviceSide.tag.level == deviceSide2.tag.level)
						{
							linkForNetworkGraph.category = "sameLayer";
						}
					}
					if (!flag)
					{
						list4.Add(linkForNetworkGraph);
					}
				}
				nodes = nodes.Where((DeviceForNetworkGraph tt) => tt.category != "SiteContainer" || (tt.category == "SiteContainer" && SiteForLinks.Any((string a) => a == tt.site))).ToList();
				list4 = list4.OrderBy((LinkForNetworkGraph c) => c.site).ToList();
				if (nodes.Count == 0)
				{
					await GetResultErrorAsync("cmdb.networkNotDevice", "هنوز اطلاعات لینک\u200cهای تجهیزات این شبکه ثبت نشده است!");
				}
				else
				{
					List<DeviceForNetworkGraph> list5 = nodes.Where((DeviceForNetworkGraph tt) => tt.category == "SiteContainer").ToList();
					foreach (DeviceForNetworkGraph siteItem in list5)
					{
						List<LinkForNetworkGraph> siteRelations = list4.Where((LinkForNetworkGraph tt) => tt.to == siteItem.key || tt.from == siteItem.key).ToList();
						int num2 = nodes.Where((DeviceForNetworkGraph tt) => siteRelations.Any((LinkForNetworkGraph a) => a.from == tt.key) || siteRelations.Any((LinkForNetworkGraph a) => a.to == tt.key)).Max((DeviceForNetworkGraph b) => b.layer);
						siteItem.layer = num2 + 1;
						List<LinkForNetworkGraph> list6 = list4.Where((LinkForNetworkGraph tt) => tt.to == siteItem.key).ToList();
						if (list6.Count == 0)
						{
							siteItem.visible = false;
						}
					}
					List<DeviceForNetworkGraph> list7 = nodes.Where((DeviceForNetworkGraph tt) => tt.category == "cloud").ToList();
					foreach (DeviceForNetworkGraph item in list7)
					{
						List<LinkForNetworkGraph> cloudRelations = list4.Where((LinkForNetworkGraph tt) => tt.to == item.key || tt.from == item.key).ToList();
						List<DeviceForNetworkGraph> list8 = nodes.Where((DeviceForNetworkGraph tt) => tt.category != "SiteContainer" && (cloudRelations.Any((LinkForNetworkGraph a) => a.from == tt.key) || cloudRelations.Any((LinkForNetworkGraph a) => a.to == tt.key))).ToList();
						int num3 = 0;
						if (list8.Count > 0)
						{
							num3 = list8.Max((DeviceForNetworkGraph b) => b.layer);
						}
						item.layer = num3 + 1;
						List<LinkForNetworkGraph> list9 = list4.Where((LinkForNetworkGraph tt) => tt.to == item.name).ToList();
						bool flag2 = false;
						foreach (LinkForNetworkGraph linkToCloud in list9)
						{
							List<DeviceForNetworkGraph> list10 = nodes.Where((DeviceForNetworkGraph tt) => tt.key == linkToCloud.from && tt.visible && tt.category != "cloud" && tt.category != "SiteContainer").ToList();
							if (list10.Count > 0)
							{
								flag2 = true;
							}
							List<DeviceForNetworkGraph> list11 = nodes.Where((DeviceForNetworkGraph tt) => tt.key == linkToCloud.from && tt.visible && tt.category != "cloud" && tt.category == "SiteContainer").ToList();
							if (list11.Count > 0)
							{
								linkToCloud.visible = false;
							}
						}
						if (!flag2)
						{
							item.visible = false;
							list4.RemoveAll((LinkForNetworkGraph tt) => tt.to == item.name);
						}
					}
					list4.RemoveAll((LinkForNetworkGraph tt) => !tt.visible);
					list4.RemoveAll((LinkForNetworkGraph tt) => nodes.Any((DeviceForNetworkGraph a) => !a.visible && a.key == tt.from));
					nodes = (from tt in nodes
						where tt.visible
						select tt into c
						orderby c.site
						select c).ThenByDescending((DeviceForNetworkGraph tt) => tt.layer).ToList();
					resultOption.status = 1;
					resultOption.result = new
					{
						nodeDataArray = nodes,
						linkDataArray = list4
					};
				}
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "GetGraphNetwork():1238");
			resultOption.message = ex.Message;
		}
		return Json(resultOption);
	}

	internal DeviceForNetworkGraph SetDeviceNode(Devices device)
	{
		DeviceForNetworkGraph deviceForNetworkGraph = new DeviceForNetworkGraph();
		deviceForNetworkGraph.key = (string.IsNullOrEmpty(device.name) ? "Unknown?" : device.name);
		deviceForNetworkGraph.name = (string.IsNullOrEmpty(device.name) ? "Unknown?" : device.name);
		deviceForNetworkGraph.ipManagement = device.ipManagement;
		deviceForNetworkGraph.type = device.type;
		deviceForNetworkGraph.model = device.series;
		deviceForNetworkGraph.rack = device.view_locations;
		deviceForNetworkGraph.site = device.site;
		deviceForNetworkGraph.deviceTag = ((device.tag.tags == null) ? string.Empty : string.Join(",", device.tag.tags.ToArray()));
		deviceForNetworkGraph.group = device.site;
		deviceForNetworkGraph.layer = device.tag.level;
		deviceForNetworkGraph.osVersion = device.osVersion;
		deviceForNetworkGraph.ownerGroup = device.ownerGroup;
		deviceForNetworkGraph.localUsername = device.localUsername;
		deviceForNetworkGraph.counter = 1;
		return deviceForNetworkGraph;
	}

	internal List<string> GetServices(List<CustomerService> customerServices)
	{
		List<string> list = new List<string>();
		foreach (CustomerService customerService in customerServices)
		{
			foreach (IDName service in customerService.services)
			{
				if (!list.Contains(service.name))
				{
					list.Add(service.name);
				}
			}
		}
		return list;
	}

	internal DeviceForNetworkGraph SetDeviceNode(Devices device, List<CustomerService> customerServices, ref List<DeviceForNetworkGraph> nodes, string currentNetwork, List<string> siteList, string deviceTag)
	{
		if (device == null)
		{
			return new DeviceForNetworkGraph();
		}
		if (string.Equals(device.tag.network, currentNetwork, StringComparison.OrdinalIgnoreCase))
		{
			string deviceKey = (string.IsNullOrEmpty(device.name) ? "Unknown?" : device.name);
			DeviceForNetworkGraph deviceForNetworkGraph = null;
			List<DeviceForNetworkGraph> list = nodes.Where((DeviceForNetworkGraph tt) => tt.key == deviceKey && tt.site == device.site).ToList();
			if (list.Count > 0)
			{
				deviceForNetworkGraph = list[0];
				deviceForNetworkGraph.services = GetServices(customerServices);
			}
			else if (siteList.Count > 0)
			{
				if (siteList.Contains(device.site))
				{
					deviceForNetworkGraph = SetDeviceNodeDetails(device, customerServices, ref nodes);
				}
				else
				{
					string siteName = device.site;
					deviceForNetworkGraph = nodes.Where((DeviceForNetworkGraph tt) => tt.key == siteName).SingleOrDefault();
					if (deviceForNetworkGraph == null)
					{
						if (string.Equals(currentNetwork, "nibn_hq", StringComparison.OrdinalIgnoreCase))
						{
							deviceForNetworkGraph = null;
						}
						else
						{
							deviceForNetworkGraph = new DeviceForNetworkGraph();
							deviceForNetworkGraph.key = siteName;
							deviceForNetworkGraph.name = "site: " + siteName;
							deviceForNetworkGraph.type = "SiteContainer";
							deviceForNetworkGraph.site = device.site;
							deviceForNetworkGraph.group = device.site;
							deviceForNetworkGraph.layer = -1;
							deviceForNetworkGraph.category = "SiteContainer";
							deviceForNetworkGraph.services = GetServices(customerServices);
							nodes.Add(deviceForNetworkGraph);
						}
					}
					else
					{
						deviceForNetworkGraph.services = GetServices(customerServices);
					}
				}
			}
			else if (string.IsNullOrEmpty(deviceTag))
			{
				deviceForNetworkGraph = SetDeviceNodeDetails(device, customerServices, ref nodes);
			}
			else if (device.tag.tags.Contains(deviceTag))
			{
				deviceForNetworkGraph = SetDeviceNodeDetails(device, customerServices, ref nodes);
			}
			else
			{
				string network = device.tag.network;
				deviceForNetworkGraph = nodes.Where((DeviceForNetworkGraph tt) => tt.key == network).SingleOrDefault();
				if (deviceForNetworkGraph == null)
				{
					if (currentNetwork.ToLower() == "nibn_hq")
					{
						deviceForNetworkGraph = null;
					}
					else
					{
						deviceForNetworkGraph = new DeviceForNetworkGraph();
						deviceForNetworkGraph.key = network;
						deviceForNetworkGraph.name = "other " + network + " devices ";
						deviceForNetworkGraph.type = "SiteContainer";
						deviceForNetworkGraph.site = device.site;
						deviceForNetworkGraph.group = network;
						deviceForNetworkGraph.layer = -1;
						deviceForNetworkGraph.category = "SiteContainer";
						deviceForNetworkGraph.services = GetServices(customerServices);
						nodes.Add(deviceForNetworkGraph);
					}
				}
				else
				{
					deviceForNetworkGraph.services = GetServices(customerServices);
				}
			}
			if (deviceForNetworkGraph != null && string.Equals(deviceForNetworkGraph.type, "Firewall ASA", StringComparison.OrdinalIgnoreCase))
			{
				int num = deviceForNetworkGraph.name.IndexOf("[");
				if (num > 0)
				{
					deviceForNetworkGraph.name = deviceForNetworkGraph.name.Substring(0, num);
				}
			}
			return deviceForNetworkGraph;
		}
		if (!string.Equals(currentNetwork, "nibn_hq", StringComparison.OrdinalIgnoreCase) || siteList.Count <= 0)
		{
			string cloudName = (string.IsNullOrEmpty(device.tag.network) ? "Other Network" : device.tag.network);
			DeviceForNetworkGraph deviceForNetworkGraph2 = nodes.Where((DeviceForNetworkGraph tt) => string.Equals(tt.key, cloudName, StringComparison.OrdinalIgnoreCase)).SingleOrDefault();
			if (deviceForNetworkGraph2 == null)
			{
				deviceForNetworkGraph2 = new DeviceForNetworkGraph();
				deviceForNetworkGraph2.key = cloudName;
				deviceForNetworkGraph2.name = cloudName;
				deviceForNetworkGraph2.type = "Cloud";
				deviceForNetworkGraph2.site = device.site;
				deviceForNetworkGraph2.layer = -1;
				deviceForNetworkGraph2.category = "cloud";
				deviceForNetworkGraph2.services = GetServices(customerServices);
				nodes.Add(deviceForNetworkGraph2);
			}
			else
			{
				deviceForNetworkGraph2.services = GetServices(customerServices);
			}
			return deviceForNetworkGraph2;
		}
		return null;
	}

	internal DeviceForNetworkGraph SetDeviceNodeDetails(Devices device, List<CustomerService> customerServices, ref List<DeviceForNetworkGraph> nodes)
	{
		DeviceForNetworkGraph deviceForNetworkGraph = null;
		string deviceKey = (string.IsNullOrEmpty(device.name) ? "Unknown?" : device.name);
		List<DeviceForNetworkGraph> list = nodes.Where((DeviceForNetworkGraph tt) => tt.name == deviceKey).ToList();
		if (list == null || list.Count == 0)
		{
			deviceForNetworkGraph = SetDeviceNode(device);
			deviceForNetworkGraph.services = GetServices(customerServices);
			nodes.Add(deviceForNetworkGraph);
		}
		else
		{
			deviceForNetworkGraph = SetDeviceNode(device);
			deviceForNetworkGraph.services = GetServices(customerServices);
			int num = list.Max((DeviceForNetworkGraph tt) => tt.counter).ToInt();
			num = (deviceForNetworkGraph.counter = num + 1);
			deviceForNetworkGraph.key = $"{deviceForNetworkGraph.key}({deviceForNetworkGraph.counter.ToString()})";
			nodes.Add(deviceForNetworkGraph);
		}
		return deviceForNetworkGraph;
	}

	internal Devices GetDeviceSide(ObjectId devIntID, out Devint devint)
	{
		devint = new Devint();
		Devices devices = _cachProvider._AllDevices.Where((Devices tt) => tt.devint.Any((Devint b) => b.id == devIntID)).SingleOrDefault();
		if (devices != null)
		{
			devint = devices.devint.Where((Devint tt) => tt.id == devIntID).SingleOrDefault();
			return devices;
		}
		return null;
	}

	[HttpGet]
	[BreadCrumb(Title = "Sites", Order = 2, GlyphIcon = "icon icon-map2 ")]
	[AddHeader("Sites")]
	[Authorize(Roles = "CMDB,Site")]
	public IActionResult Sites()
	{
		return View();
	}

	[HttpGet]
	[BreadCrumb(Title = "SitesPlan", Order = 2, GlyphIcon = "icon icon-cube ")]
	[AddHeader("SitesPlan")]
	[Authorize(Roles = "CMDB,Site")]
	public IActionResult SitesPlan()
	{
		List<Sites> list = _cachProvider._AllSites.Where((Sites tt) => tt.siteDrawInfo != null).ToList();
		List<SiteBuilding> list2 = new List<SiteBuilding>();
		SiteBuilding siteBuilding = null;
		foreach (Sites site in list)
		{
			siteBuilding = list2.Where((SiteBuilding tt) => tt.Main == site.mainName).SingleOrDefault();
			if (siteBuilding == null)
			{
				siteBuilding = new SiteBuilding();
				siteBuilding.Main = site.mainName;
				siteBuilding.Sites = new Dictionary<string, string>();
				siteBuilding.Sites.Add(site.id.ToString(), $"{site.main}-{site.secondary}");
				list2.Add(siteBuilding);
			}
			else if (!siteBuilding.Sites.ContainsKey(site.id.ToString()))
			{
				siteBuilding.Sites.Add(site.id.ToString(), $"{site.main}-{site.secondary}");
			}
		}
		base.ViewData["Sites"] = list2;
		return View();
	}

	[HttpPost]
	[AjaxOnly]
	[Authorize(Roles = "CMDB")]
	public async Task<IActionResult> GetSites()
	{
		try
		{
			await AddEventLogAsync("CMDB-Show-Sites", base.CurrentUser.mainUserInfo.UserName);
			return new JsonResult(_cachProvider._AllSites.Select((Sites site) => new Sites
			{
				id = site.id,
				siteDrawInfo = site.siteDrawInfo,
				main = site.main,
				mainName = site.mainName,
				secondary = site.secondary,
				secondaryName = site.secondaryName,
				building = site.building,
				view_rack_count = site.view_rack_count
			}).ToList());
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetSites():85");
		}
		return new JsonResult(null);
	}

	[HttpPost]
	[AjaxOnly]
	public JsonResult GetRacksBySite([FromForm] GridParam gridParam)
	{
		JqGridResponse jqGridResponse = new JqGridResponse();
		try
		{
			string site = string.Empty;
			List<string> racksBySiteID = GetRacksBySiteID(ObjectId.Parse(gridParam.id), out site);
			if (racksBySiteID != null)
			{
				jqGridResponse = new JqGridResponse
				{
					TotalPagesCount = 1,
					PageIndex = 0,
					TotalRecordsCount = racksBySiteID.Count()
				};
				foreach (string item in racksBySiteID)
				{
					string text = item.ToString().Replace(" ", "_").Replace("(", "")
						.Replace(")", "")
						.Replace(".", "");
					jqGridResponse.Records.Add(new JqGridRecord(Convert.ToString(text), new RackView
					{
						id = text,
						graph = string.Empty,
						siteId = gridParam.id,
						siteName = site,
						device_count = GetDevicesBySite_Rack(site, item).Count,
						name = item
					}));
				}
				jqGridResponse.Reader.RepeatItems = false;
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetRacksBySite():1117");
		}
		return new JqGridJsonResult(jqGridResponse);
	}

	[HttpPost]
	[AjaxOnly]
	public JsonResult GetDevicesByRack([FromForm] GridParam gridParam)
	{
		JqGridResponse jqGridResponse = new JqGridResponse();
		try
		{
			string rack = gridParam.id.Substring(gridParam.id.IndexOf("_") + 1, gridParam.id.Length - gridParam.id.IndexOf("_") - 1);
			gridParam.id = gridParam.id.Substring(0, gridParam.id.IndexOf("_"));
			string siteBySiteID = GetSiteBySiteID(ObjectId.Parse(gridParam.id));
			List<Devices> devicesBySite_Rack = GetDevicesBySite_Rack(siteBySiteID, rack);
			if (devicesBySite_Rack != null)
			{
				jqGridResponse = new JqGridResponse
				{
					TotalPagesCount = 1,
					PageIndex = 0,
					TotalRecordsCount = devicesBySite_Rack.Count()
				};
				foreach (Devices item in devicesBySite_Rack)
				{
					jqGridResponse.Records.Add(new JqGridRecord(Convert.ToString(item.id), new Devices
					{
						id = item.id,
						name = item.name,
						type = item.type,
						gridview_network = item.tag.network,
						ipManagement = item.ipManagement,
						series = item.series,
						model = item.model,
						view_devint_count = item.view_devint_count,
						view_patch_count = item.view_patch_count,
						site = item.site,
						view_locations = item.view_locations,
						gridview_networkLevel = item.tag.level,
						backupDevice = item.backupDevice,
						powerBackup = item.powerBackup,
						isVirtual = item.isVirtual,
						localUsername = item.localUsername,
						ownerGroup = item.ownerGroup
					}));
				}
				jqGridResponse.Reader.RepeatItems = false;
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetDevicesByRack():1174");
		}
		return new JqGridJsonResult(jqGridResponse);
	}

	[HttpPost]
	[Authorize(Roles = "CMDB")]
	public IActionResult GetLinksByDevice1([FromForm] GridParam gridParam)
	{
		JqGridResponse jqGridResponse = new JqGridResponse();
		try
		{
			List<Devint> interfacesByDeviceID = GetInterfacesByDeviceID(ObjectId.Parse(gridParam.id));
			List<ObjectId> devintIDs = new List<ObjectId>(interfacesByDeviceID.Select((Devint s) => s.id).ToList());
			IMongoCollection<Links> collection = _mongoDbContext.GetCollection<Links>("Links");
			List<Links> allLinks = (from tt in collection.AsQueryable()
				where devintIDs.Contains(tt.side1) || devintIDs.Contains(tt.side2)
				select tt).ToList();
			List<DeviceLink> list = new List<DeviceLink>();
			foreach (Devint item in interfacesByDeviceID)
			{
				list.AddRange(GetLinkInterface(item, allLinks));
			}
			if (list != null)
			{
				jqGridResponse = new JqGridResponse
				{
					TotalPagesCount = 1,
					PageIndex = 0,
					TotalRecordsCount = list.Count()
				};
				foreach (DeviceLink item2 in list)
				{
					jqGridResponse.Records.Add(new JqGridRecord(Convert.ToString(item2.id), new DeviceLink
					{
						id = item2.id,
						sourceType = item2.sourceType,
						sourceNumber = item2.sourceNumber,
						sourcePortChannel = item2.sourcePortChannel,
						sourceIP = item2.sourceIP,
						sourceState = item2.sourceState,
						sourceStyle = item2.sourceStyle,
						sourceTransportMode = item2.sourceTransportMode,
						sourceVlanList = item2.sourceVlanList,
						name = item2.name,
						type = item2.type,
						number = item2.number,
						portChannel = item2.portChannel,
						linkType = item2.linkType,
						networks = item2.networks,
						lastUpdateName = item2.lastUpdateName,
						lastUpdateSystem = item2.lastUpdateSystem,
						lastUpdateDate = item2.lastUpdateDate,
						info = item2.info.Replace("<", "(").Replace(">", ")")
					}));
				}
				jqGridResponse.Reader.RepeatItems = false;
			}
		}
		catch (Exception exception)
		{
			_logger.LogCritical(exception, "GetLinksByDevice1():490");
		}
		return new JqGridJsonResult(jqGridResponse);
	}

	[HttpPost]
	[AjaxOnly]
	public async Task<JsonResult> GetGraphSite([FromForm] GraphSite graphSite)
	{
		_ = 1;
		try
		{
			if (string.IsNullOrEmpty(graphSite.name))
			{
				await GetResultErrorAsync("cmdb.nullSite", "اطلاعات کامل به سمت سرور ارسال نشده است!");
			}
			else
			{
				List<string> racks = new List<string>();
				string SiteName = string.Empty;
				if (graphSite.isRack)
				{
					SiteName = GetSiteBySiteID(ObjectId.Parse(graphSite.id));
					racks.Add(graphSite.name.ToString().ToUpper());
				}
				else
				{
					racks = GetRacksBySiteID(ObjectId.Parse(graphSite.id), out SiteName);
				}
				await AddEventLogAsync("CMDB-Show-GraphSite", "Site:" + SiteName);
				List<Devices> devicesByRacks = GetDevicesByRacks(SiteName, racks);
				List<DeviceForRackGraph> list = new List<DeviceForRackGraph>();
				DeviceForRackGraph deviceForRackGraph = new DeviceForRackGraph();
				deviceForRackGraph.key = (string.IsNullOrEmpty(SiteName) ? graphSite.name.ToString() : SiteName);
				deviceForRackGraph.site = SiteName;
				deviceForRackGraph.type = "Site";
				list.Add(deviceForRackGraph);
				foreach (string rack in racks)
				{
					deviceForRackGraph = new DeviceForRackGraph();
					deviceForRackGraph.key = rack;
					deviceForRackGraph.site = SiteName;
					deviceForRackGraph.type = "Rack";
					list.Add(deviceForRackGraph);
					IEnumerable<Devices> enumerable = from tt in devicesByRacks
						where tt.locations.Any((DeviceLocation a) => a.rack == rack)
						select (tt);
					if (enumerable == null || enumerable.Count() <= 0)
					{
						continue;
					}
					foreach (Devices item in enumerable)
					{
						foreach (DeviceLocation location in item.locations)
						{
							if (location.rack != rack)
							{
								break;
							}
							deviceForRackGraph = new DeviceForRackGraph();
							deviceForRackGraph.key = item.name;
							deviceForRackGraph.site = SiteName;
							deviceForRackGraph.type = item.type;
							deviceForRackGraph.model = item.series;
							deviceForRackGraph.rack = location.rack;
							deviceForRackGraph.unit = location.startUnit;
							deviceForRackGraph.height = location.endUnit - location.startUnit;
							List<ObjectId> devintIDs = item.devint.Select((Devint a) => a.id).ToList();
							IMongoCollection<Links> collection = _mongoDbContext.GetCollection<Links>("Links");
							List<Links> source = (from tt in collection.AsQueryable()
								where devintIDs.Contains(tt.side1) || devintIDs.Contains(tt.side1)
								select tt).ToList();
							List<List<CustomerService>> list2 = source.Select((Links tt) => tt.customerServices).ToList();
							List<string> list3 = new List<string>();
							foreach (List<CustomerService> item2 in list2)
							{
								foreach (CustomerService item3 in item2)
								{
									foreach (IDName service in item3.services)
									{
										if (!list3.Contains(service.name))
										{
											list3.Add(service.name);
										}
									}
								}
							}
							deviceForRackGraph.services = list3;
							list.Add(deviceForRackGraph);
						}
					}
				}
				resultOption.status = 1;
				resultOption.result = new
				{
					siteName = SiteName,
					devices = list
				};
			}
		}
		catch (Exception ex)
		{
			_logger.LogCritical(ex, "GetGraphSite():756");
			resultOption.message = ex.Message;
		}
		return Json(resultOption);
	}

	[HttpPost]
	[AjaxOnly]
	public async Task<JsonResult> GetGraphLocation([FromForm] GridParam gridParam)
	{
		try
		{
			Sites item = _cachProvider._AllSites.FirstOrDefault((Sites tt) => tt.id == ObjectId.Parse(gridParam.id));
			if (item == null || item.siteDrawInfo == null)
			{
				await GetResultErrorAsync("cmdb.siteId", "کد سایت اشتباه ارسال شده است!");
			}
			else
			{
				await AddEventLogAsync("CMDB-Show-SitePlan", $"Site: {item.main}-{item.secondary}");
				GetResultOk();
				resultOption.result = item.siteDrawInfo;
			}
		}
		catch (Exception ex)
		{
			resultOption.message = ex.Message;
			_logger.LogCritical(ex, "GetGraphLocation():774");
		}
		return Json(resultOption);
	}

	[HttpPost]
	[AjaxOnly]
	public async Task<JsonResult> EditGraphLocation([FromForm] GraphSiteLocation siteLocation)
	{
		_ = 2;
		try
		{
			ObjectId id = ((siteLocation.id != null) ? ObjectId.Parse(siteLocation.id) : ObjectId.Parse("000000000000000000000000"));
			List<NodeData> list = new List<NodeData>();
			if (!string.IsNullOrEmpty(siteLocation.nodeData))
			{
				list = JsonConvert.DeserializeObject<List<NodeData>>(siteLocation.nodeData);
			}
			List<LinkData> list2 = new List<LinkData>();
			if (!string.IsNullOrEmpty(siteLocation.linkData))
			{
				list2 = JsonConvert.DeserializeObject<List<LinkData>>(siteLocation.linkData);
			}
			Sites sites = _cachProvider._AllSites.FirstOrDefault((Sites tt) => tt.id == id);
			if (sites == null || sites.siteDrawInfo == null)
			{
				await GetResultErrorAsync("cmdb.siteId", "کد سایت اشتباه ارسال شده است!");
			}
			else
			{
				UpdateDefinition<Sites> update = Builders<App.Core.Collection.Sites>.Update.Set((Sites x) => x.siteDrawInfo.nodeData, list).Set((Sites x) => x.siteDrawInfo.linkData, list2);
				IMongoCollection<Sites> collection = _mongoDbContext.GetCollection<Sites>("Sites");
				collection.UpdateOne((Sites x) => x.id == id, update);
				sites.siteDrawInfo.nodeData = list;
				sites.siteDrawInfo.linkData = list2;
				await AddEventLogAsync("CMDB-Edit-Location", $"Site: {sites.main}-{sites.secondary}");
				await GetResultOkAsync("cmdb.siteEdit", "اطلاعات سایت با موفقیت به روز رسانی شد");
			}
		}
		catch (Exception ex)
		{
			resultOption.message = ex.Message;
			_logger.LogCritical(ex, "EditGraphLocation():833");
		}
		return Json(resultOption);
	}

	[HttpPost]
	[AjaxOnly]
	public JsonResult GetRackDevices([FromForm] RakDeviceParam rakParam)
	{
		try
		{
			ObjectId id = ((rakParam.id != null) ? ObjectId.Parse(rakParam.id.ToString()) : ObjectId.Parse("000000000000000000000000"));
			string text = ((rakParam.rak != null) ? rakParam.rak : "");
			string siteBySiteID = GetSiteBySiteID(id);
			List<Devices> devicesBySite_Rack = GetDevicesBySite_Rack(siteBySiteID, text);
			List<DeviceForRackShow> list = new List<DeviceForRackShow>();
			DeviceForRackShow deviceForRackShow = null;
			if (devicesBySite_Rack != null)
			{
				resultOption.status = 1;
				foreach (Devices item in devicesBySite_Rack)
				{
					foreach (DeviceLocation location in item.locations)
					{
						if (!(location.rack != text))
						{
							deviceForRackShow = new DeviceForRackShow();
							deviceForRackShow.key = item.name;
							deviceForRackShow.ipManagement = item.ipManagement;
							deviceForRackShow.startUnit = location.startUnit;
							deviceForRackShow.height = location.endUnit - location.startUnit;
							deviceForRackShow.rackFront = string.Equals(location.rackFront, "front", StringComparison.OrdinalIgnoreCase);
							deviceForRackShow.type = item.type;
							deviceForRackShow.series = (string.IsNullOrEmpty(item.series) ? item.model : item.series);
							deviceForRackShow.localUsername = item.localUsername;
							deviceForRackShow.ownerGroup = item.ownerGroup;
							deviceForRackShow.osVersion = item.osVersion;
							deviceForRackShow.label = item.label;
							deviceForRackShow.propertyNo = item.propertyNo;
							list.Add(deviceForRackShow);
							continue;
						}
						break;
					}
				}
				resultOption.result = list;
			}
		}
		catch (Exception ex)
		{
			resultOption.message = ex.Message;
			_logger.LogCritical(ex, "GetRackDevices():877");
		}
		return Json(resultOption);
	}

	internal List<string> GetRacksBySiteID(ObjectId Id, out string site)
	{
		site = string.Empty;
		Sites sites = _cachProvider._AllSites.Where((Sites tt) => tt.id == Id).SingleOrDefault();
		if (sites != null && sites.racks != null && sites.racks.Count > 0)
		{
			site = sites.main;
			return sites.racks.OrderBy((string tt) => tt).ToList().ConvertAll((string d) => d.ToUpper());
		}
		return new List<string>();
	}

	internal string GetSiteBySiteID(ObjectId Id)
	{
		Sites sites = _cachProvider._AllSites.FirstOrDefault((Sites tt) => tt.id == Id);
		if (sites != null)
		{
			return sites.main;
		}
		return string.Empty;
	}

	internal List<Devices> GetDevicesBySite_Rack(string site, string rack)
	{
		IQueryable<Devices> queryable = ((!base.User.IsInRole("Site")) ? MyDevices.Where((Devices tt) => tt.site.ToUpper() == site.ToUpper() && tt.locations.Any((DeviceLocation a) => a.rack == rack.ToUpper())).AsQueryable() : _cachProvider._AllDevices.Where((Devices tt) => tt.site.ToUpper() == site.ToUpper() && tt.locations.Any((DeviceLocation a) => a.rack == rack.ToUpper())).AsQueryable());
		if (queryable != null)
		{
			return (from tt in queryable.ToList()
				orderby tt.locations
				select tt).ToList();
		}
		return new List<Devices>();
	}

	internal List<Devices> GetDevicesByRacks(string site, List<string> racks)
	{
		IQueryable<Devices> queryable = ((!base.User.IsInRole("Site")) ? MyDevices.Where((Devices tt) => tt.site.ToUpper() == site.ToUpper() && tt.locations.Any((DeviceLocation a) => racks.Contains(a.rack))).AsQueryable() : _cachProvider._AllDevices.Where((Devices tt) => tt.site.ToUpper() == site.ToUpper() && tt.locations.Any((DeviceLocation a) => racks.Contains(a.rack))).AsQueryable());
		if (queryable != null)
		{
			return queryable.ToList();
		}
		return new List<Devices>();
	}
}
