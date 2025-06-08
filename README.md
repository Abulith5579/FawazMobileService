FawazMobileService
using System;
using System.Diagnostics;
using System.IO.Ports;
using System.Windows.Forms;

namespace FawazMobileService
{
    static class Program
    {
        [STAThread]
        static void Main()
        {
            Application.EnableVisualStyles();
            Application.SetCompatibleTextRenderingDefault(false);
            Application.Run(new MainForm());
        }
    }

    public class MainForm : Form
    {
        private ComboBox cmbPorts;
        private Button btnCheckPort;
        private Button btnCheckAdb;
        private Button btnOpenTools;
        private TextBox txtAdbResult;

        public MainForm()
        {
            this.Text = "Fawaz Mobile Service";
            this.Width = 600;
            this.Height = 450;
            this.StartPosition = FormStartPosition.CenterScreen;

            Label lblPort = new Label { Text = "Select COM Port:", Left = 50, Top = 30, Width = 100 };
            cmbPorts = new ComboBox { Left = 160, Top = 25, Width = 200 };
            btnCheckPort = new Button { Text = "Refresh Ports", Left = 380, Top = 25, Width = 100 };

            btnCheckAdb = new Button { Text = "Check ADB Devices", Left = 160, Top = 70, Width = 200 };
            btnOpenTools = new Button { Text = "Open Tools", Left = 160, Top = 110, Width = 200 };

            txtAdbResult = new TextBox { Left = 50, Top = 150, Width = 520, Height = 220, Multiline = true, ScrollBars = ScrollBars.Vertical, ReadOnly = true };

            btnCheckPort.Click += BtnCheckPort_Click;
            btnCheckAdb.Click += BtnCheckAdb_Click;
            btnOpenTools.Click += BtnOpenTools_Click;

            Controls.Add(lblPort);
            Controls.Add(cmbPorts);
            Controls.Add(btnCheckPort);
            Controls.Add(btnCheckAdb);
            Controls.Add(btnOpenTools);
            Controls.Add(txtAdbResult);

            RefreshPorts();
        }

        private void RefreshPorts()
        {
            cmbPorts.Items.Clear();
            string[] ports = SerialPort.GetPortNames();
            foreach (string port in ports)
                cmbPorts.Items.Add(port);
            if (ports.Length > 0)
                cmbPorts.SelectedIndex = 0;
        }

        private void BtnCheckPort_Click(object sender, EventArgs e)
        {
            RefreshPorts();
            if (cmbPorts.Items.Count == 0)
                MessageBox.Show("No COM ports found.", "Info", MessageBoxButtons.OK, MessageBoxIcon.Information);
        }

        private void BtnCheckAdb_Click(object sender, EventArgs e)
        {
            try
            {
                ProcessStartInfo psi = new ProcessStartInfo
                {
                    FileName = "adb",
                    Arguments = "devices",
                    RedirectStandardOutput = true,
                    UseShellExecute = false,
                    CreateNoWindow = true
                };
                using (Process process = Process.Start(psi))
                {
                    string output = process.StandardOutput.ReadToEnd();
                    process.WaitForExit();
                    txtAdbResult.Text = output;
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show("ADB Error: " + ex.Message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnOpenTools_Click(object sender, EventArgs e)
        {
            ToolsForm toolsForm = new ToolsForm(cmbPorts.SelectedItem?.ToString());
            toolsForm.ShowDialog();
        }
    }

    public class ToolsForm : Form
    {
        private Button btnCDMA;
        private Button btnBaseband;
        private Button btnForce4G;
        private string selectedPort;

        public ToolsForm(string comPort)
        {
            selectedPort = comPort;

            this.Text = "Tools - Fawaz Mobile Service";
            this.Width = 400;
            this.Height = 300;
            this.StartPosition = FormStartPosition.CenterScreen;

            btnCDMA = new Button { Text = "🔄 Convert to CDMA", Left = 75, Top = 30, Width = 250, Height = 40 };
            btnBaseband = new Button { Text = "🛠️ Fix Baseband", Left = 75, Top = 80, Width = 250, Height = 40 };
            btnForce4G = new Button { Text = "📶 Force 4G", Left = 75, Top = 130, Width = 250, Height = 40 };

            btnCDMA.Click += (s, e) => ExecuteSerialCommand("AT+CTEC=1");
            btnBaseband.Click += (s, e) => ExecuteAdbCommand("shell su -c 'setprop persist.radio.baseband MDM'");
            btnForce4G.Click += (s, e) => ExecuteSerialCommand("AT+QCFG=\"nwscanmode\",3,1");

            Controls.Add(btnCDMA);
            Controls.Add(btnBaseband);
            Controls.Add(btnForce4G);
        }

        private void ExecuteAdbCommand(string command)
        {
            try
            {
                ProcessStartInfo psi = new ProcessStartInfo
                {
                    FileName = "adb",
                    Arguments = command,
                    RedirectStandardOutput = true,
                    UseShellExecute = false,
                    CreateNoWindow = true
                };
                using (Process process = Process.Start(psi))
                {
                    string output = process.StandardOutput.ReadToEnd();
                    process.WaitForExit();
                    MessageBox.Show("ADB Result:\n" + output, "ADB Output", MessageBoxButtons.OK, MessageBoxIcon.Information);
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show("ADB Error: " + ex.Message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void ExecuteSerialCommand(string command)
        {
            if (string.IsNullOrEmpty(selectedPort))
            {
                MessageBox.Show("No COM port selected!", "Error", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            try
            {
                using (SerialPort port = new SerialPort(selectedPort, 115200))
                {
                    port.Open();
                    port.WriteLine(command + "\r\n");
                    MessageBox.Show("Sent to Serial (" + selectedPort + "): " + command, "Serial Port", MessageBoxButtons.OK, MessageBoxIcon.Information);
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show("Serial Error: " + ex.Message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }
}