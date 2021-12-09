---
title: "c#从命令中获取输出实时显示在控件中 "
date: 2021-12-09T10:58:02+08:00
draft: false
tags: ["C#", "WinForms"]
---

A brief description of what the code performs in this example:

The shell command (cmd.exe) is run first, using `start /WAIT` as parameter. More or less the same functionality as `/k` : the console is started without any specific task, waiting to process a command when one is sent.

[StandardOutput](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process.standardoutput?view=netframework-4.7.2#System_Diagnostics_Process_StandardOutput), [StandardError](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process.standarderror?view=netframework-4.7.2#System_Diagnostics_Process_StandardError) and [StandardInput](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process.standardinput?view=net-6.0) are all redirected, setting [RedirectStandardOutput](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.processstartinfo.redirectstandardoutput?view=net-6.0), [RedirectStandardError](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.processstartinfo.redirectstandarderror?view=net-6.0) and [RedirectStandardInput](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.processstartinfo.redirectstandardinput?view=net-6.0) properties of the [ProcessStartInfo](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.processstartinfo?view=net-6.0) to true.
<!-- more -->

The console Output stream, when written to, will raise the [OutputDataReceived](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process.outputdatareceived?view=net-6.0) event; it's content can be read from the e.Data member of the [DataReceivedEventArgs](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.datareceivedeventargs?view=net-6.0).
StandardError will use its ErrorDataReceived event for the same purpose.
You could use a single event handler for both the events, but, after some testing, you might realize that is probably not a good idea. Having them separated avoids some weird overlapping and allows to easily tell apart errors from normal output (as a note, you can find programs that write to the error Stream instead of the output Stream).

`StandardInput` can be redirected assigning it to a [StreamWriter](https://docs.microsoft.com/en-us/dotnet/api/system.io.streamwriter?view=net-6.0) stream.
Each time a string is written to the stream, the console will interpret that input as a command to be executed.

Also, the Process is instructed to rise it's Exited event upon termination, setting its [EnableRaisingEvents](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process.enableraisingevents?view=net-6.0) property to true.
The [Exited](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process.exited?view=net-6.0) event is raised when the Process is closed because an Exit command is processed or calling the .[Close()](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process.close?view=net-6.0) method (or, eventually, the [.Kill()](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process.kill?view=net-6.0) method, which should only be used when a Process is not responding anymore, for some reason).

Since we need to pass the console Output to some UI controls (RichTextBoxes in this example) and the Process events are raised in ThreadPool Threads, we must synchronize this context with the UI's.
This can be done using the Process [SynchronizingObject](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process.synchronizingobject?view=net-6.0) property, setting it to the Parent Form or using the [Control.BeginInvoke](https://docs.microsoft.com/en-us/dotnet/api/system.windows.forms.control.begininvoke) method, that will execute a delegate function on the thread where the control's handle belongs.
Here, a [MethodInvoker](https://docs.microsoft.com/en-us/dotnet/api/system.windows.forms.methodinvoker) representing the delegate is used for this purpose.

The core function used to instantiate the Process and set its properties and event handlers:

```c#
using System;
using System.Diagnostics;
using System.IO;
using System.Windows.Forms;

public partial class frmCmdInOut : Form
{
    Process cmdProcess = null;
    StreamWriter stdin = null;

    public frmCmdInOut() => InitializeComponent();

    private void MainForm_Load(object sender, EventArgs e)
    {
        rtbStdIn.Multiline = false;
        rtbStdIn.SelectionIndent = 20;
    }

    private void btnStartProcess_Click(object sender, EventArgs e)
    {
        btnStartProcess.Enabled = false;
        StartCmdProcess();
        btnEndProcess.Enabled = true;
    }

    private void btnEndProcess_Click(object sender, EventArgs e)
    {
        if (stdin.BaseStream.CanWrite) {
            stdin.WriteLine("exit");
        }
        btnEndProcess.Enabled = false;
        btnStartProcess.Enabled = true;
        cmdProcess?.Close();
    }

    private void rtbStdIn_KeyPress(object sender, KeyPressEventArgs e)
    {
        if (e.KeyChar == (char)Keys.Enter) {
            if (stdin == null) {
                rtbStdErr.AppendText("Process not started" + Environment.NewLine);
                return;
            }

            e.Handled = true;
            if (stdin.BaseStream.CanWrite) {
                stdin.Write(rtbStdIn.Text + Environment.NewLine);
                stdin.WriteLine();
                // To write to a Console app, just 
                // stdin.WriteLine(rtbStdIn.Text); 
            }
            rtbStdIn.Clear();
        }
    }

    private void StartCmdProcess()
    {
        var pStartInfo = new ProcessStartInfo {
             FileName = "cmd.exe", 
            // Batch File Arguments = "/C START /b /WAIT somebatch.bat", 
            // Test: Arguments = "START /WAIT /K ipconfig /all", 
            Arguments = "START /WAIT", 
            WorkingDirectory = Environment.SystemDirectory, 
            // WorkingDirectory = Application.StartupPath, 
            RedirectStandardOutput = true, 
            RedirectStandardError = true, 
            RedirectStandardInput = true, 
            UseShellExecute = false, 
            CreateNoWindow = true, 
            WindowStyle = ProcessWindowStyle.Hidden, 
        };

        cmdProcess = new Process {
            StartInfo = pStartInfo, 
            EnableRaisingEvents = true, 
            // Test without and with this
            // When SynchronizingObject is set, no need to BeginInvoke()
            //SynchronizingObject = this
        };

        cmdProcess.Start();
        cmdProcess.BeginErrorReadLine();
        cmdProcess.BeginOutputReadLine();
        stdin = cmdProcess.StandardInput;
        // stdin.AutoFlush = true;  <- already true

        cmdProcess.OutputDataReceived += (s, evt) => {
            if (evt.Data != null)
            {
                BeginInvoke(new MethodInvoker(() => {
                    rtbStdOut.AppendText(evt.Data + Environment.NewLine);
                    rtbStdOut.ScrollToCaret();
                }));
            }
        };

        cmdProcess.ErrorDataReceived += (s, evt) => {
            if (evt.Data != null) {
                BeginInvoke(new Action(() => {
                    rtbStdErr.AppendText(evt.Data + Environment.NewLine);
                    rtbStdErr.ScrollToCaret();
                }));
            }
        };

        cmdProcess.Exited += (s, evt) => {
            stdin?.Dispose();
            cmdProcess?.Dispose();
        };
    }
}
```

Since the StandardInput has been redirected to a StreamWriter:

```c#
stdin = cmdProcess.StandardInput;
```

we just write to the Stream to execute a command:

```c#
stdin.WriteLine(["Command Text"]);
```

> From: https://stackoverflow.com/questions/51680382/how-do-i-get-output-from-a-command-to-appear-in-a-control-on-a-form-in-real-time




