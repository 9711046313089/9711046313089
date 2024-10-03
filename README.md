public class TaskItem
{
    public int Id { get; set; } // Unique identifier
    public string Title { get; set; } // Task title
    public string Description { get; set; } // Task description
    public DateTime DueDate { get; set; } // Task due date
    public bool Completed { get; set; } // Status of the task
    public string Priority { get; set; } // Priority: low, medium, high
    public string Recurrence { get; set; } // Recurrence: daily, weekly, monthly
}
using Microsoft.EntityFrameworkCore;
using TaskManagerAPI.Models;

namespace TaskManagerAPI.Data
{
    public class TaskContext : DbContext
    {
        public TaskContext(DbContextOptions<TaskContext> options) : base(options) { }
        public DbSet<TaskItem> Tasks { get; set; }
    }
}
public class TaskDbContext : DbContext
{
    public TaskDbContext(DbContextOptions<TaskDbContext> options) : base(options) { }
    public DbSet<Task> Tasks { get; set; }
}
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<TaskDbContext>(options =>
        options.UseNpgsql(Configuration.GetConnectionString("DefaultConnection")));

    services.AddHangfire(x => x.UseSqlServerStorage(Configuration.GetConnectionString("DefaultConnection")));
    services.AddHangfireServer();
    
    services.AddControllers();
}
[Route("api/[controller]")]
[ApiController]
public class TasksController : ControllerBase
{
    private readonly TaskDbContext _context;

    public TasksController(TaskDbContext context)
    {
        _context = context;
    }

    // POST /tasks
    [HttpPost]
    public async Task<IActionResult> CreateTask([FromBody] Task task)
    {
        _context.Tasks.Add(task);
        await _context.SaveChangesAsync();
        return Ok(task);
    }

    // GET /tasks
    [HttpGet]
    public async Task<IActionResult> GetAllTasks()
    {
        var tasks = await _context.Tasks.ToListAsync();
        return Ok(tasks);
    }

    // GET /tasks/{id}
    [HttpGet("{id}")]
    public async Task<IActionResult> GetTaskById(int id)
    {
        var task = await _context.Tasks.FindAsync(id);
        if (task == null) return NotFound();
        return Ok(task);
    }

    // PUT /tasks/{id}
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateTask(int id, [FromBody] Task taskUpdate)
    {
        var task = await _context.Tasks.FindAsync(id);
        if (task == null) return NotFound();

        task.Title = taskUpdate.Title;
        task.Description = taskUpdate.Description;
        task.DueDate = taskUpdate.DueDate;
        task.Completed = taskUpdate.Completed;
        task.Priority = taskUpdate.Priority;
        task.Recurrence = taskUpdate.Recurrence;

        await _context.SaveChangesAsync();
        return Ok(task);
    }

    // DELETE /tasks/{id}
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteTask(int id)
    {
        var task = await _context.Tasks.FindAsync(id);
        if (task == null) return NotFound();

        _context.Tasks.Remove(task);
        await _context.SaveChangesAsync();
        return Ok();
    }
}
[HttpPost("suggest")]
public IActionResult SuggestTask([FromBody] string input)
{
    var suggestions = new List<string>
    {
        "Meeting with client",
        "Finish project report",
        "Prepare presentation"
    };

    var result = suggestions.Where(s => s.Contains(input, StringComparison.OrdinalIgnoreCase)).ToList();
    return Ok(result);
}
[HttpPost("{id}/predict-due-date")]
public async Task<IActionResult> PredictDueDate(int id)
{
    var completedTasks = await _context.Tasks
        .Where(t => t.Completed)
        .ToListAsync();

    if (!completedTasks.Any()) return Ok(DateTime.UtcNow.AddDays(2)); // Default 2 days

    var avgDuration = completedTasks.Average(t => (t.DueDate - DateTime.UtcNow).TotalDays);
    var predictedDate = DateTime.UtcNow.AddDays(avgDuration);

    return Ok(predictedDate);
}
public class TaskReminderService
{
    private readonly TaskDbContext _context;

    public TaskReminderService(TaskDbContext context)
    {
        _context = context;
    }

    public void SendReminders()
    {
        var upcomingTasks = _context.Tasks
            .Where(t => t.DueDate <= DateTime.UtcNow.AddHours(24) && !t.Completed)
            .ToList();

        foreach (var task in upcomingTasks)
        {
            Console.WriteLine($"Reminder: Task '{task.Title}' is due soon.");
        }
    }
}
public void Configure(IApplicationBuilder app, IBackgroundJobClient backgroundJobs)
{
    backgroundJobs.Enqueue(() => Console.WriteLine("Background Job started!"));
    RecurringJob.AddOrUpdate<TaskReminderService>("task-reminders", service => service.SendReminders(), Cron.Hourly);
    
    app.UseHangfireDashboard();
    app.UseHangfireServer();
}
public void CreateRecurringTasks()
{
    var recurringTasks = _context.Tasks
        .Where(t => t.Recurrence != null)
        .ToList();

    foreach (var task in recurringTasks)
    {
        Task newTask = task.Clone();  // Assuming a Clone method to duplicate task
        switch (task.Recurrence)
        {
            case "daily":
                newTask.DueDate = task.DueDate.AddDays(1);
                break;
            case "weekly":
                newTask.DueDate = task.DueDate.AddDays(7);
                break;
            case "monthly":
                newTask.DueDate = task.DueDate.AddMonths(1);
                break;
        }
        _context.Tasks.Add(newTask);
    }
    _context.SaveChanges();
} 
