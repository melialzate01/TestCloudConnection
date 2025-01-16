namespace TestCloudConnection_1
{
    using System;
    using System.Collections.Generic;
    using System.Globalization;
    using System.Linq;
    using System.Text;
    using Skyline.DataMiner.Automation;
    using Skyline.DataMiner.Core.DataMinerSystem.Automation;
    using Skyline.DataMiner.DcpNodeHelper.Common;
    using Skyline.DataMiner.Utils.InteractiveAutomationScript;

    /// <summary>
    /// Represents a DataMiner Automation script.
    /// </summary>
    public class Script
    {
        //// engine.ShowUI();
        private InteractiveController app;
        private IEngine engine;
        private CloudConnectionDialog dialog;

        public void Run(IEngine engine)
        {
            try
            {
                this.engine = engine;
                app = new InteractiveController(engine);

                engine.SetFlag(RunTimeFlags.NoKeyCaching);
                engine.Timeout = TimeSpan.FromHours(10);

                RunSafe();
            }
            catch (Exception ex)
            {
                engine.Log($"Run|Something went wrong: {ex}");
                ShowExceptionDialog(ex);
            }
        }

        private void RunSafe()
        {
            dialog = new CloudConnectionDialog(engine);

            // Assign a single event handler for all buttons
            dialog.CheckButton.Pressed += (sender, args) => HandleButtonPressed("Check");
            dialog.ResetButton.Pressed += (sender, args) => HandleButtonPressed("Reset");
            dialog.CancelButton.Pressed += (sender, args) => HandleButtonPressed("Cancel");

            app.Run(dialog);
        }

        private void HandleButtonPressed(string buttonType)
        {
            switch (buttonType)
            {
                case "Check":
                    string orgId = dialog.OrgIdInput.Text;
                    string coordId = dialog.CoordIdInput.Text;

                    if (string.IsNullOrWhiteSpace(orgId) || string.IsNullOrWhiteSpace(coordId))
                    {
                        dialog.ResultLabel.Text = "Please enter valid Cloud Organization Id and Coordination Id.";
                        dialog.ResultLabel.IsVisible = true;
                        return;
                    }

                    try
                    {
                        NodeHelperBuilder builder = new NodeHelperBuilder();
                        var endpoint = builder.Build();
                        //engine.GenerateInformation($"orgId: {orgId} coordId: {coordId}");
                        var connectionInfo = endpoint.GetDmsConnectionInfo(orgId, coordId);
                        dialog.ResultLabel.Text = connectionInfo.HasValidConnection
                            ? "Cloud connection is valid."
                            : "Cloud connection is invalid.";
                        dialog.ResultLabel.IsVisible = true;

                        //foreach (var connectionInfo in connectionInfos)
                        //{
                        //    dialog.ResultLabel.Text = connectionInfo.HasValidConnection
                        //        ? "Cloud connection is valid."
                        //        : "Cloud connection is invalid.";
                        //    dialog.ResultLabel.IsVisible = true;
                        //}
                    }
                    catch (Exception ex)
                    {
                        dialog.ResultLabel.Text = $"Error retrieving connection info: {ex.Message}";
                        dialog.ResultLabel.IsVisible = true;
                    }
                    break;

                case "Reset":
                    dialog.OrgIdInput.Text = string.Empty;
                    dialog.CoordIdInput.Text = string.Empty;
                    dialog.ResultLabel.Text = "Result will be displayed here.";
                    dialog.ResultLabel.IsVisible = false;
                    break;

                case "Cancel":
                    engine.ExitSuccess("Canceled by user");
                    break;

                default:
                    dialog.ResultLabel.Text = "Unknown action.";
                    dialog.ResultLabel.IsVisible = true;
                    break;
            }
        }

        private void ShowExceptionDialog(Exception exception)
        {
            var exceptionDialog = new ExceptionDialog(engine, exception);
            exceptionDialog.OkButton.Pressed += (sender, args) => engine.ExitFail("Something went wrong.");
            if (app.IsRunning) app.ShowDialog(exceptionDialog); else app.Run(exceptionDialog);
        }
    }

    public class CloudConnectionDialog : Dialog
    {
        public CloudConnectionDialog(IEngine engine) : base(engine)
        {
            Title = "Check Cloud Connection Status";

            OrgIdLabel = new Label("Cloud Organization Id:");
            CoordIdLabel = new Label("Cloud Coordination Id:");
            OrgIdInput = new TextBox();
            CoordIdInput = new TextBox();
            CheckButton = new Button("Check Connection");
            ResetButton = new Button("Reset");
            CancelButton = new Button("Cancel");
            ResultLabel = new Label("Result will be displayed here.") { IsVisible = false };

            AddWidget(OrgIdLabel, 0, 0);
            AddWidget(OrgIdInput, 0, 1, 1, 2); // Span two columns for alignment
            AddWidget(CoordIdLabel, 1, 0);
            AddWidget(CoordIdInput, 1, 1, 1, 2); // Span two columns for alignment

            // Align buttons
            AddWidget(CheckButton, 2, 0);
            AddWidget(ResetButton, 2, 1);
            AddWidget(CancelButton, 2, 2, 1, 1); // Align Cancel button correctly

            AddWidget(ResultLabel, 3, 0, 1, 3); // Span across 3 columns for full width
        }

        public Label OrgIdLabel { get; }
        public Label CoordIdLabel { get; }
        public TextBox OrgIdInput { get; }
        public TextBox CoordIdInput { get; }
        public Button CheckButton { get; }
        public Button ResetButton { get; }
        public Button CancelButton { get; }
        public Label ResultLabel { get; }
    }
}
