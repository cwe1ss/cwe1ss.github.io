---
title: Find projects which are missing in your "All Projects" solution
date: 2014-08-21T00:00:00+01:00
categories:
- .NET
toc:
  enable: false
---

Do you use a Visual Studio solution which contains all of your projects to do daily builds? If you have lots of projects and if many people are involved it's very likely that somebody forgets to add his project to this solution. 

This small program helps you by showing you all csproj-files that are not part of your solution file!

```c#
class Program
{
    static void Main(string[] args)
    {
        // Parameters
        string baseFolder = @"C:\path\to\solution\";
        string slnFile = "AllProjects.sln";
        string outputFile = "MissingProjects.txt";

        string slnContent = File.ReadAllText(Path.Combine(baseFolder, slnFile));

        string[] projectFiles = Directory.GetFiles(baseFolder, "*.csproj", SearchOption.AllDirectories);

        List<string> missingProjects = projectFiles
            .Where(fullPath => slnContent.IndexOf(fullPath.Replace(baseFolder, ""), StringComparison.OrdinalIgnoreCase) < 0)
            .ToList();

        File.WriteAllLines(outputFile, missingProjects);

        Console.WriteLine("Projects missing in solution: " + missingProjects.Count);
        Console.WriteLine("Details: " + outputFile);
        Console.ReadLine();
    }
}
```