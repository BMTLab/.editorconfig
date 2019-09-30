# Example
                
```
#region HEADER
// MieltaConfiguratorWeb HomeController.cs.cs
// Created by Nikita Neverov at 24.09.2019 12:13
#endregion


#define ADD_INITAL

using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Runtime.CompilerServices;
using System.Text;
using System.Threading.Tasks;

using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Localization;
using Microsoft.Extensions.Logging;

using MieltaConfiguratorWeb.Data;
using MieltaConfiguratorWeb.Models;
using MieltaConfiguratorWeb.ViewModels;

using MieltaNetworkDriver;
using MieltaNetworkDriver.Host;
using MieltaNetworkDriver.Host.Exceptions;


namespace MieltaConfiguratorWeb.Controllers
{
    public class HomeController : Controller
    {
        #region Fields
        internal Host ConfiguratorTcpHost;

        private readonly TrackerConfiguratorWebContext _context;
        private readonly IStringLocalizer<HomeController> _localizer;
        private readonly ILogger<HomeController> _logger;
        #endregion _Fields

        #region Constructors
        public HomeController
        (
            ILogger<HomeController> logger,
            IStringLocalizer<HomeController> localizer
        )
        {
            #region InitializeDI
            _context = new TrackerConfiguratorWebContext();
            _localizer = localizer;
            _logger = logger;
            #endregion


            #if ADD_INITAL
            var mielta = new User { Name = "ООО НПО МИЭЛТА" };

            _context.Trackers.AddRange(
                new Tracker
                {
                    Imei = "865401039574144",
                    Model = "M7",
                    FirmwareVersion = "dev0.1",
                    User = mielta
                },
                new Tracker
                {
                    Imei = "865401039574151",
                    Model = "M3",
                    FirmwareVersion = "dev0.2",
                    User = mielta
                },
                new Tracker
                {
                    Imei = "867567320091876",
                    Model = "M1",
                    FirmwareVersion = "dev0.3",
                    User = mielta
                },
                new Tracker
                {
                    Imei = "777567320091876",
                    Model = "M1",
                    FirmwareVersion = "dev0.3",
                    User = mielta
                });

            _context.SaveChangesAsync();
            #endif // _ADD_INITAL

            RunTcpHostAsync();
        }
        #endregion _Constructors


        #region Methods
        [NonAction]
        private async void RunTcpHostAsync()
        {
            try
            {
                ConfiguratorTcpHost = new TcpHost()
                                     .SetIp(NetworkHelper.GetOwnIp())
                                     .SetPort(44355)
                                     .SetBufferSize(64)
                                     .SetCapacityOfConnections(16)
                                     .SetConcurencyLevelOfConnections(2)
                                     .SetValidateKeyHandler(imei => imei.Length == 15)
                                     .Create();

                _logger.LogTrace(@"Tcp host have been created");

                ConfiguratorTcpHost.ActiveConnections.ConnectionAdded += async (_, e) =>
                {
                    await using var c = new TrackerConfiguratorWebContext();

                    if (!TrackerExists(e.Imei))
                        return;

                    await c.ConnectionLogs.AddAsync((e.Imei, e.ConnectionDateTime,
                                                     e.TcpClient.Client.RemoteEndPoint));

                    await c.SaveChangesAsync().ConfigureAwait(false);

                    var listCommand =
                        await c.Commands
                               .Where(t => t.TrackerImei == e.Imei)
                               .ToListAsync()
                               .ConfigureAwait(false) ??
                        new List<Command>(0);

                    if (listCommand.Count == 0)
                        return;

                    var sb = new StringBuilder(32, 256);
                    sb.AppendJoin(';', listCommand);

                    var stream = e.TcpClient.GetStream();
                    var sData = sb.ToString();
                    var bData = Encoding.Unicode.GetBytes(sData);

                    if (stream.CanWrite)
                        await stream
                             .WriteAsync(bData, 0, bData.Length)
                             .ConfigureAwait(false);

                    stream.Flush();
                    stream.Close();
                    e.TcpClient?.Close();
                };

                ConfiguratorTcpHost.HostStarted += (_, e) =>
                {
                    _logger.LogTrace("Host started");
                };

                ConfiguratorTcpHost.HostStopped += (_, e) =>
                {
                    _logger.LogTrace("Host stopped");
                };

                await ConfiguratorTcpHost
                     .StartAsync()
                     .ConfigureAwait(false);
            }
            catch (HostException hex)
            {
                _logger.LogError(hex.Message);
            }
            finally
            {
                ConfiguratorTcpHost?.Stop();
            }
        }


        #region Methods.HTTP
        // GET: Home
        public async Task<IActionResult> Index()
        {
            ViewData["Title"] = _localizer["lc_trackers"];

            await using var c = new TrackerConfiguratorWebContext();

            return View(await c.Trackers
                               .Include(t => t.User)
                               .ToListAsync()
                               .ConfigureAwait(false));
        }


        // GET: Home/Create/
        public async Task<IActionResult> Create()
        {
            await using var c = new TrackerConfiguratorWebContext();
            ViewBag.Users = c.Users.ToList();

            return View();
        }


        // POST: Home/Create
        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> Create(Tracker tracker, int? userId)
        {
            await using var c = new TrackerConfiguratorWebContext();

            if (string.IsNullOrEmpty(tracker?.Imei))
                return NotFound();

            if (TrackerExists(tracker.Imei))
                return View();

            tracker.User = await c.Users.FindAsync(userId);

            await c.Trackers.AddAsync(tracker);

            await c.SaveChangesAsync()
                   .ConfigureAwait(false);

            return RedirectToAction(nameof(Index));
        }


        // GET: Home/Delete
        public async Task<IActionResult> Delete(string id)
        {
            if (string.IsNullOrEmpty(id))
                return NotFound();

            if (!TrackerExists(id))
                return NotFound();

            await using var c = new TrackerConfiguratorWebContext();

            var tracker = (await c.Trackers
                                  .Include(t => t.User)
                                  .ToListAsync()
                                  .ConfigureAwait(false))
               .Find(t => t.Imei == id);

            return View(tracker);
        }


        // POST: Home/Delete
        [HttpPost]
        [ActionName("Delete")]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> DeleteConfirmed(string id)
        {
            await using var c = new TrackerConfiguratorWebContext();

            c.Trackers.Remove(await c.Trackers
                                     .FindAsync(id));

            await c.SaveChangesAsync()
                   .ConfigureAwait(false);

            return RedirectToAction(nameof(Index));
        }


        // GET: Home/Details/
        public async Task<IActionResult> Details(string id)
        {
            if (string.IsNullOrEmpty(id))
                return NotFound();

            if (!TrackerExists(id))
                return NotFound();

            await using var c = new TrackerConfiguratorWebContext();

            var tracker = (await c.Trackers
                                  .Include(t => t.User)
                                  .ToListAsync()
                                  .ConfigureAwait(false))
               .Find(t => t.Imei == id);

            return View(tracker);
        }


        // GET: Home/Edit/
        public async Task<IActionResult> Edit(string id)
        {
            if (string.IsNullOrEmpty(id))
                return NotFound();

            if (!TrackerExists(id))
                return NotFound();

            await using var c = new TrackerConfiguratorWebContext();

            var tracker = (await c.Trackers
                                  .Include(t => t.User)
                                  .ToListAsync()
                                  .ConfigureAwait(false))
               .Find(t => t.Imei == id);

            return View(new Command { Tracker = tracker, TrackerImei = tracker.Imei });
        }


        // POST: Home/Edit
        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> Edit(Command command)
        {
            await using var c = new TrackerConfiguratorWebContext();

            await c.Commands.AddAsync(command);

            await c.SaveChangesAsync().ConfigureAwait(false);

            return RedirectToAction(nameof(Index));
        }


        // GET: Home/Privacy
        public IActionResult Privacy()
        {
            return View();
        }


        // GET: Error
        [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
        public IActionResult Error()
        {
            return View(new ErrorViewModel
            {
                RequestId = Activity.Current?.Id ??
                            HttpContext.TraceIdentifier
            });
        }
        #endregion _Methods.HTTP


        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        private bool TrackerExists(string imei)
        {
            return _context.Trackers.Any(e => e.Imei == imei);
        }
        #endregion _Methods
    }
}
```
                
				
### END
