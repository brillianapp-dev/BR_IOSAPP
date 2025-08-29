using BRN2Sys.BrCommand.Board.Util;
using BRN2Sys.Objects;
using System;
using System.Collections.Generic;
using System.Data;
using System.Drawing;
using System.IO;
using System.IO.Ports;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace BRN2Sys
{
    public partial class ucTLP : UserControl
    {
        public enum eAI
        {
            Inlet = 1,
            Outlet = 2,
            Flow = 3,
            Flow2 = 4,
            AirCurtain_Inlet = 4,
            AirCurtain_Flow = 5
        }
        public enum eDI
        {
            FoupType = 0x01,
            InterLockedByPass = 0x04,
            DoorOpen = 0x08,
            CarrierLock = 0x10,
            Presence = 0x20,
            OperationalStatus = 0x40,
            Placement = 0x80,
            Clamp_R_ON = 0x40,
            Clamp_L_ON = 0x20,
            Clamp_R_OFF = 0x08,
            Clamp_L_OFF = 0x02
        }
        public enum eDO
        {
            AllOFF = 0x00,
            Inlet_ON = 0x01,
            Inlet_OFF = 0x02,
            Outlet = 0x04,
            AirCurtain = 0x10,
            Inlet_H = 0x01,
            Inlet_L = 0x08,
            NozzleControl = 0x08,
            Clamp = 0x20
        }

        enum eCommanT_Alarm
        {
            OK = 0x00,
            Lenght_Error = 0xF1,
            CheckSum_Error = 0xF2,
            ToolID_Error = 0xF4,
            TimeOut_Error = 0xE1,
            Value_Error = 0xE2,
            DataConvert_Error = 0xE4,
            Error = 0xFF
        }

        public enum eStep
        {
            Idle = 0,
            LoadPurge = 1,
            ContinuePurge = 2,
            UnloadPurge = 3,
            PurgeFinish = 8
        }

        public enum eProcessStep
        {
            None,
            Idle,
            Idle_Initial,
            Idle_PurgeStart,
            Idle_Purging,
            Idle_PurgeEnd,
            Load_Initial,
            Load_PurgeStart,
            Load_Purging,
            Load_PurgeEnd,
            Continue_Initial,
            Continue_PurgeStart,
            Continue_Purging,
            Continue_PurgeEnd,
            Unload_Initial,
            Unload_PurgeStart,
            Unload_Purging,
            Unload_PurgeEnd,
            //Leak_Initial,
            //Leak_CheckStart,
            //Leak_Checking,
            //Leak_CheckEnd,
            //Post_Initial,
            //Post_PurgeStart,
            //Post_Purging,
            //Post_PurgeEnd,
            PurgeFinish
        }

        public enum eUISide { Left, Right }

        public cPar_Port par_Port { get; set; }
        public cPar_General par_General { get; set; }
        public cSecs cSecsManager { get; set; }

        // public event EventHandler<string> ErrorEventHandle;
        SerialPort serialPort;
        List<byte> receiveDataList = new List<byte>();
        Queue<byte[]> sendDataList = new Queue<byte[]>();
        byte[] sendData;
        bool _AutoRunFlag = false;
        string _tid = "";
        double _InletPress = 0;
        double _OutletPress = 0;
        double _Humidity = 0;
        double _Flow = 0;
        double _Temperature = 0;
        double _AirCurtainInletPress = 0;
        double _AirCurtainFlow = 0;
        bool _InterLock = false;
        byte _di = 0;
        byte _do = 0;
        float[] lAi = new float[6];
        float[] outletArray_Buffer = new float[40];
        int outletArray_Index = 0;
        float[] humidityArray_Buffer = new float[10];
        int humiditytArray_Index = 0;


        public eStep SysStep { get; set; }
        private eStep SysStep_Temp;
        public eProcessStep AutoRun_Step { get; set; }
        private eProcessStep AutoRun_Step_Temp;
        DateTime currentTimeTick = DateTime.Now;

        eUISide uISide = eUISide.Left;
        DateTime time_PurgeStart = DateTime.Now;
        DateTime time_ManualModeStart = DateTime.Now;//for auto change mode

        DateTime time_FoupInStatusNotMatch = DateTime.Now;
        byte CheckSumErrorCount = 0;


        string _AlarmMessage = string.Empty;
        bool _AlarmEnable = false;
        DateTime _AlarmStartTime = DateTime.Now;

        int time_ManualModeTimeout = 300;//auto manual to auto mode (second)
        int time_PurgeLimit = 0;
        int time_MBBoardSend = Environment.TickCount;
        int time_MBBoardRecieve = Environment.TickCount;
        int count_MBBoardTimeOut = 0;

        int count_InletLikeOutlet = 0;
        bool isAlarm_BoardTimeout = false;

        bool isAlarm_TCommand = false;
        int count_TCommand = 0;
        DateTime time_TErrorCheck = DateTime.Now;

        bool isAlarm_RHError = false;
        DateTime time_RHErrorCheck = DateTime.Now;
        int count_RHError = 0;


        bool _FoupInStatus = false;
        DateTime time_FoupChangeStart = DateTime.Now.AddDays(-1);


        bool isPostIdlePurge = false;

        bool needDoAutoRecover = false;

        #region property
        ucTLP_Status _PortStatus = new ucTLP_Status();
        public ucTLP_Status PortStatus { get { return _PortStatus; } }
        public string TID { get { return _tid; } }


        public DateTime ManualModeStartTime
        {
            get { return time_ManualModeStart; }
            set { time_ManualModeStart = value; }
        }

        public double InletPress
        {
            get
            {
                _InletPress = (lAi[(int)eAI.Inlet] - par_Port.iPxInAiSubVal) / par_Port.iPxInAiDivVal;
                return Math.Round(_InletPress + par_Port.dPxInAiOffsetVal, 3);
            }
        }
        public double OutletPress
        {
            get
            {
                if (par_General.FgUseMovingAverageOutletPress)
                {
                    var sortArray = outletArray_Buffer.ToList();
                    sortArray.Sort();
                    if (sortArray.Count <= par_General.iOutletRemoveHeadIte + par_General.iOutletRemoveEndItem)
                    {
                        return 0;
                    }
                    var avg = sortArray.Skip(par_General.iOutletRemoveHeadIte).Take(sortArray.Count - par_General.iOutletRemoveHeadIte - par_General.iOutletRemoveEndItem).Average();
                    var resultAVG = (Math.Round(avg, 1)) * -1;//outlet need reverse
                    //var resultAVG = (Math.Round(outletArray_Buffer.Average(), 1)) * -1;//outlet need reverse
                    if (resultAVG > 0 && par_General.FgOutletPressureOnlyZeroShow)
                    { return 0; }
                    return resultAVG;
                }

                _OutletPress = (lAi[(int)eAI.Outlet] - par_Port.iPxOutAiSubVal) * 100 / par_Port.iPxOutAiDivVal;
                var result = (Math.Round(_OutletPress + par_Port.dPxOutAiOffsetVal, 1)) * -1;//outlet need reverse
                if (result > 0 && par_General.FgOutletPressureOnlyZeroShow)
                { return 0; }
                return result;
            }
        }
        public double Humidity//0.1~99.99
        {
            get
            {
                if (par_General.FgHumidityUseArrayAvg)
                {

                    var resultAVG = Math.Round(humidityArray_Buffer.Average(), 2);
                    if (resultAVG < 0.1) { resultAVG = 0.1; }
                    return resultAVG;
                }

                var result = _Humidity + par_Port.dPxHumidityOffsetVal;
                if (result < 0.1)
                { result = 0.1; }
                else if (result > 99.99)
                { result = 99.99; }
                return result;
            }
        }
        public double Flow //0~300
        {
            get
            {
                _Flow = (lAi[(int)eAI.Flow] - par_Port.iPxFlowAiSubVal) / par_Port.iPxFlowAiDivVal;
                var result = Math.Round(_Flow + par_Port.dPxFlowOffsetVal, 0);
                if (result < 0)
                { result = 0; }
                else if (result > 300)
                { result = 300; }
                return result;
            }
        }
        public double Temperature //0.1~99.99
        {
            get
            {
                var result = _Temperature + par_Port.dPxTemperatureOffsetVal;
                if (result < 0.1)
                { result = 0.1; }
                else if (result > 99.99)
                { result = 99.99; }
                return result;
            }
        }
        public bool AirCurtainEnable
        {
            get
            {
                if (par_General.FgAirCurtainFunction && par_Port.FgAirCurtainEnabled)
                { return true; }
                else
                { return false; }
            }
        }

        public double AirCurtainInletPress
        {
            get
            {
                if (!AirCurtainEnable)
                { return 0; }
                _AirCurtainInletPress = (lAi[(int)eAI.AirCurtain_Inlet] - par_Port.iPxPressAiSubValAirCurtain) / par_Port.dPxPressAiDivValAirCurtain;
                return Math.Round(_AirCurtainInletPress + par_Port.dPxAirCurtainInAiOffsetVal, 3);
            }
        }
        public double AirCurtainFlow //0~500
        {
            get
            {
                if (!AirCurtainEnable)
                { return 0; }
                _AirCurtainFlow = (lAi[(int)eAI.AirCurtain_Flow] - par_Port.iPxFlowAiSubValAirCurtain) / par_Port.dPxFlowAiDivValAirCurtain;

                var result = Math.Round(_AirCurtainFlow + par_Port.dPxAirCurtainFlowOffsetVal, 0);
                if (result < 0)
                { result = 0; }
                else if (result > 500)
                { result = 500; }
                return result;
            }
        }

        public bool InterLock { get { return _InterLock; } }

        public bool isAutoMode
        {
            get { return par_Port.FgAutoMode; }
            set
            {
                if (!value)
                { time_ManualModeStart = DateTime.Now; }
                if (par_Port.FgAutoMode != value)
                {
                    string mode = value ? "Auto Mode" : "Manual Mode";
                    cBrLog.WriteFileLog(par_Port.PortName, cBrLog.eLogType.System, $"Mode change to {mode}.");
                    if (value)
                    { AutoRun_Idle(); }
                }
                par_Port.FgAutoMode = value;
            }
        }
        public bool DisableCOM
        {
            get { return par_Port.FgPxComDisable; }
            set
            {
                try
                {
                    if (!SerialPort.GetPortNames().Contains(serialPort.PortName))
                    { return; }
                    if (value)
                    {
                        if (serialPort.IsOpen)
                        {
                            serialPort.Close();
                            cBrLog.WriteFileLog(par_Port.PortName, cBrLog.eLogType.System, $"COM Port({par_Port.PortName}) Turn OFF.");
                        }
                    }
                    else
                    {
                        time_MBBoardSend = time_MBBoardRecieve = Environment.TickCount;
                        if (!serialPort.IsOpen)
                        {
                            if (sendDataList.Count > 0)
                            {
                                sendDataList.Clear();
                            }
                            serialPort.Open();
                            cBrLog.WriteFileLog(par_Port.PortName, cBrLog.eLogType.System, $"COM Port({par_Port.PortName}) Turn ON.");

                            // SetDO_InitialType();//for com port open set MB initial DO type
                        }
                    }
                }
                catch (Exception ex)
                {
                    cBrLog.WriteFileLog(par_Port.PortName, cBrLog.eLogType.Warning, $"DisableCOM exception {ex.Message}.");
                }
                par_Port.FgPxComDisable = value;
            }
        }

        public bool ComStatus { get { return serialPort.IsOpen; } }

        public string FoupID { get; set; } = "";
        public bool FoupInStatus { get { return _FoupInStatus; } set { _FoupInStatus = value; } }
        public string AlarmMessage { get { return _AlarmMessage; } set { _AlarmMessage = value; } }
        public eALID AlarmID { get; set; }
        //string _AlarmMessage = string.Empty;
        //bool _AlarmEnable = false;
        //DateTime _AlarmStartTime = DateTime.Now;
        public bool AlarmEnable
        {
            get { return _AlarmEnable; }
            set
            {
                _AlarmEnable = value;
                if (value)
                { _AlarmStartTime = DateTime.Now; }
            }
        }
        public DateTime AlarmStartTime { get { return _AlarmStartTime; } }

        public eUISide DI_side { get { return uISide; } set { uISide = value; } }

        public string Board_Version { get; set; }
        public string Board_CheckSum { get; set; }

        public string UserID { get; set; }


        #endregion property

        public ucTLP(cPar_Port par_Port, cPar_General par_General, cSecs cSecs = null)
        {
            InitializeComponent();
            this.par_Port = par_Port;
            this.par_General = par_General;
            cSecsManager = cSecs;
            SysStep = SysStep_Temp = eStep.Idle;
            AutoRun_Step = AutoRun_Step_Temp = eProcessStep.None;
            _tid = "0" + par_Port.PortName[par_Port.PortName.Length - 1];
            FoupID = UserID = "";
            serialPort = new SerialPort($"COM{par_Port.ReaderLocalPort}", 57600, Parity.None, 8, StopBits.One);
            serialPort.DataReceived += SerialPort_DataReceived;
            serialPort.ReadTimeout = 5000;
            serialPort.WriteTimeout = 5000;
            if (SerialPort.GetPortNames().Contains(serialPort.PortName) && !serialPort.IsOpen && !DisableCOM)
            {
                try
                {
                    serialPort.Open();
                    cBrLog.WriteFileLog(par_Port.PortName, cBrLog.eLogType.System, $"COM Port({par_Port.PortName}) Turn ON.");
                }
                catch (Exception ex)
                {
                    cBrLog.WriteFileLog(par_Port.PortName, cBrLog.eLogType.Exception, $"COM Open({ex.Message})");
                }
            }


            // UIBinding();
            UpdateUI();

            //check update port status
            _PortStatus.Name = par_Port.PortName;
            _PortStatus.Warming = isAlarm_TCommand ? "FF" : "00";
            _PortStatus.DI_Status = Convert.ToString(_di, 2).PadLeft(8, '0');
            _PortStatus.DO_Status = Convert.ToString(_do, 2).PadLeft(8, '0');
            _PortStatus.RH = Humidity.ToString();
            _PortStatus.Temp = Temperature.ToString();
            _PortStatus.Flow = Flow.ToString();
            _PortStatus.Inlet = InletPress.ToString();
            _PortStatus.Outlet = OutletPress.ToString();
            _PortStatus.MID = FoupID;


            AutoRun_Start();
        }

        ~ucTLP()
        {
            //serialPort.DataReceived -= SerialPort_DataReceived;
            //if (serialPort.IsOpen)
            //{ serialPort.Close(); }

        }
        public void DisposeCom()
        {
            if (serialPort == null)
            { return; }
            if (serialPort.IsOpen)
            { serialPort.Close(); }
            serialPort.DataReceived -= SerialPort_DataReceived;
        }

        public void UpdateUI()
        {
            try
            {
                txtPortName.Text = par_Port.PortName;


                btnValve_H.Visible = picOutValve.Visible = par_General.PurgeEnable;
                btnValve_L.Visible = (par_General.FgTwoStepFlowFunction && par_General.PurgeEnable);

                btnClamp.Visible = txtClamp1.Visible = txtClamp2.Visible = par_General.FgClampControlFunction;
                txtDoorOpen.Visible = txtPresence.Visible = txtOperStatue.Visible = !par_General.FgClampControlFunction;

                btnValve_AirCurtain.Visible = AirCurtainEnable;
                txtAirCurtainFlow.Visible = txtAirCurtainFlowValue.Visible = txtAirCurtainInlet.Visible = txtAirCurtainInletValue.Visible = par_General.FgAirCurtainInletPress;
                btnGiveUpPurging.Visible = par_General.FgGiveUpPurgingButtonShow;



                if (par_General.PurgeEnable)
                {
                    btnPurge.Visible = !isAutoMode;
                }
                else
                { btnPurge.Visible = btnEndPurgeCycle.Visible = false; }



                if (isAutoMode)
                {
                    txtModeValue.Text = "AUTO";
                    txtModeValue.BackColor = Color.SlateBlue;

                }
                else
                {
                    txtModeValue.Text = "MANUAL";
                    txtModeValue.BackColor = Color.Lime;
                }


                //txtStepName.Text = SysStep == eStep.Idle ? (AutoRun_Step == eProcessStep.Idle_Purging || AutoRun_Step == eProcessStep.Leak_Checking || AutoRun_Step == eProcessStep.Post_Purging ? AutoRun_Step.ToString() : "") : SysStep.ToString();
                if (SysStep == eStep.Idle)
                {
                    if (AutoRun_Step == eProcessStep.Idle_Purging)
                    {

                        txtStepName.Text = eProcessStep.Idle_Purging.ToString();
                    }
                    else
                    { txtStepName.Text = ""; }
                }
                else
                {
                    txtStepName.Text = SysStep.ToString();
                }

                txtStepValue.Text = ((int)SysStep).ToString();
                txtAlarmValue.BackColor = _AlarmEnable ? Color.Red : Color.Lime;

                txtAlarmMessage.Text = AlarmMessage;
                txtFoupID.Text = FoupID != null ? FoupID : string.Empty;


                txtRHValue.Text = Humidity.ToString();
                txtTempValue.Text = Temperature.ToString();
                txtInletPressureValue.Text = InletPress.ToString();
                txtOutletPressureValue.Text = OutletPress.ToString();
                txtFlowValue.Text = Flow.ToString();
                txtAirCurtainInletValue.Text = AirCurtainInletPress.ToString();
                txtAirCurtainFlowValue.Text = AirCurtainFlow.ToString();

                if (serialPort.IsOpen)
                {

                    txtPortEnable.BackColor = Color.Lime;
                    txtPortEnable.Text = "Enable Port";

                    btnValve_H.BackColor = GetDO(eDO.Inlet_H) ? Color.MistyRose : Color.LightGray;
                    btnValve_L.BackColor = GetDO(eDO.Inlet_L) ? Color.MistyRose : Color.LightGray;
                    btnValve_AirCurtain.BackColor = GetDO(eDO.AirCurtain) ? Color.Lime : Color.LightGray;

                    picOutValve.Image = GetDO(eDO.Outlet) ? Properties.Resources.Valve_ON : Properties.Resources.Valve_Off;
                    picFoupType.Image = GetDI(eDI.FoupType) ? Properties.Resources.FoupType_1 : Properties.Resources.FoupType_0;

                    txtOperStatue.BackColor = GetDI(eDI.OperationalStatus) ? Color.Lime : Color.DarkGray;
                    txtPresence.BackColor = GetDI(eDI.Presence) ? Color.Lime : Color.DarkGray;
                    txtPlacement.BackColor = GetDI(eDI.Placement) ? Color.Lime : Color.DarkGray;
                    txtCarrierLocked.BackColor = GetDI(eDI.CarrierLock) ? Color.Lime : Color.DarkGray;
                    txtDoorOpen.BackColor = GetDI(eDI.DoorOpen) ? Color.Lime : Color.DarkGray;

                }
                else
                {
                    txtPortEnable.BackColor = Color.Red;
                    txtPortEnable.Text = "Disable Com";
                }

            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"UpdateUI({ex.Message})", FoupID);
                //  return false;
            }
        }

        //raise alarm and send s98f51
        private void RaiseAlarm(eALID aLID, bool alarmEnable = true)
        {
            try
            {
                var alid_new = (int)aLID+ (100*(Convert.ToInt32(TID)-1));//for same alid but different means.
                if (alarmEnable)
                {

                    AlarmEnable = alarmEnable;
                    AlarmID = (eALID)alid_new;
                    AlarmMessage = cSecsManager.GetALTX((eALID)alid_new, Convert.ToInt32(TID));


                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Alarm, AlarmMessage, FoupID);
                    cSecsManager.Send_S5F1(AlarmID, Convert.ToInt32(TID));
                }
                else
                {
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Alarm, cSecsManager.GetALTX((eALID)alid_new, Convert.ToInt32(TID)), FoupID);
                    cSecsManager.Send_S5F1((eALID)alid_new, Convert.ToInt32(TID));
                    //cSecsManager.Send_S98F51(AlarmID, Convert.ToByte(TID), FoupID);
                }

            }
            catch (Exception ex)
            { cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"ResetAlarm({ex.Message})", FoupID); }
        }


        public void ResetAlarm()
        {
            try
            {
                if (AlarmEnable)
                {
                    //clear alid
                    cSecsManager.Send_S5F1(AlarmID, Convert.ToInt32(TID),Enable:false);

                    AlarmEnable = false;
                    AlarmMessage = "";
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Do Reset Alarm.", FoupID);
                    if (needDoAutoRecover)
                    {
                        needDoAutoRecover = false;
                    }
                    time_MBBoardSend = time_MBBoardRecieve = Environment.TickCount;
                    count_MBBoardTimeOut = 0;

                    isAlarm_BoardTimeout = false;
                    count_InletLikeOutlet = 0;

                    isAlarm_TCommand = false;
                    count_TCommand = 0;

                    isAlarm_RHError = false;
                    count_RHError = 0;

                    CheckSumErrorCount = 0;
                }
            }
            catch (Exception ex)
            { cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"ResetAlarm({ex.Message})", FoupID); }
        }

        #region cmd
        private void SerialPort_DataReceived(object sender, SerialDataReceivedEventArgs e)
        {
            try
            {
                time_MBBoardRecieve = Environment.TickCount;
                byte[] bytes = new byte[serialPort.BytesToRead];
                var ReciveData = serialPort.Read(bytes, 0, bytes.Length);
                receiveDataList.AddRange(bytes);

                var commandReturn = BoardUtil.SplitBrCommand(ref receiveDataList);
                if (commandReturn.Bytes.Count > 0 && commandReturn.CheckSumResult)
                {
                    // var dataArray = commandReturn.ToArray();
                    var dataArray = commandReturn.Bytes.ToArray();

                    if (par_General.FgSave5DIOLog)
                    {
                        var _5DIOLog = BitConverter.ToString(dataArray);
                        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"Receive Data: {_5DIOLog}", FoupID);
                    }

                    switch ((BoardCommandType)dataArray[4])
                    {
                        case BoardCommandType.FirmwareCheck:
                            var length = dataArray[1] - board_util.BR_COMMAND_LEN_START_BIT;
                            Board_Version = Encoding.ASCII.GetString(dataArray.Skip(5).Take(length - 5).ToArray());
                            var CheckSum = dataArray.Skip(length).Take(2).ToArray();
                            Board_CheckSum = CheckSum[0].ToString("X2") + CheckSum[1].ToString("X2");

                            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"Board Version={Board_Version} & Board CheckSum={Board_CheckSum}", FoupID);
                            break;
                        case BoardCommandType.StatusInfo:

                            if (dataArray[5] != 0 && !isAlarm_TCommand)
                            {
                                if ((DateTime.Now - time_TErrorCheck).TotalSeconds > 1)
                                {
                                    time_TErrorCheck = DateTime.Now;
                                    GetErrorDetect();
                                    count_TCommand++;
                                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"Board FF Count={count_TCommand}", FoupID);
                                    //   break;
                                }

                            }
                            if (dataArray[5] == 0)
                            {
                                count_RHError = 0;
                                count_TCommand = 0;
                            }

                            _InterLock = dataArray[6] == 0x30 ? true : false;

                            //DI1 need reverse
                            var di1 = dataArray[7] & 0x01;
                            var diTemp = dataArray[7];
                            dataArray[7] = (byte)(((diTemp >> 1) << 1) + (di1 > 0 ? 0 : 1));

                            if (_di != dataArray[7] || _do != dataArray[8])
                            {
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"DO Now:{Convert.ToString(dataArray[8], 2).PadLeft(8, '0')} Last:{Convert.ToString(_do, 2).PadLeft(8, '0')}.DI Now:{Convert.ToString(dataArray[7], 2).PadLeft(8, '0')} Last:{Convert.ToString(_di, 2).PadLeft(8, '0')}", FoupID);

                                //add tool record
                                if ((_di & (byte)eDI.CarrierLock) == 0 && ((dataArray[7] & (byte)eDI.CarrierLock) > 0)) //carrier lock 0>>1
                                {
                                    if (par_Port.port_ToolRecord.ClampCounts > (int.MaxValue * 0.9))
                                    {
                                        par_Port.port_ToolRecord.ClampCounts = 0;
                                    }
                                    par_Port.port_ToolRecord.ClampCounts++;
                                }
                                if ((_do & (byte)eDO.Inlet_OFF) == 0 && ((dataArray[8] & (byte)eDO.Inlet_OFF) > 0)) //Inlet_Off 0>>1
                                {
                                    if (par_Port.port_ToolRecord.PurgeCounts > (int.MaxValue * 0.9))
                                    {
                                        par_Port.port_ToolRecord.PurgeCounts = 0;
                                    }
                                    par_Port.port_ToolRecord.PurgeCounts++;
                                }

                            }
                            _di = dataArray[7];
                            _do = dataArray[8];

                            _Humidity = (dataArray[9] * 256 + dataArray[10]) * 0.01;
                            _Temperature = (dataArray[11] * 256 + dataArray[12]) * 0.01;
                            lAi[1] = (dataArray[13] * 256 + dataArray[14]);
                            lAi[2] = (dataArray[15] * 256 + dataArray[16]);
                            lAi[3] = (dataArray[17] * 256 + dataArray[18]);
                            lAi[4] = (dataArray[19] * 256 + dataArray[20]);
                            lAi[5] = (dataArray[21] * 256 + dataArray[22]);

                            if (par_General.FgHumidityUseArrayAvg)
                            {
                                if (humidityArray_Buffer.Length != par_General.iHumidityArrayCount)
                                {
                                    humidityArray_Buffer = new float[par_General.iHumidityArrayCount];
                                    humiditytArray_Index = 0;
                                }
                                humidityArray_Buffer[humiditytArray_Index] = (float)(_Humidity + par_Port.dPxHumidityOffsetVal);
                                //HumiditytArray_Index++;
                                humiditytArray_Index = humiditytArray_Index >= par_General.iHumidityArrayCount - 1 ? 0 : humiditytArray_Index + 1;
                            }
                            if (par_General.FgUseMovingAverageOutletPress)
                            {
                                if (outletArray_Buffer.Length != par_General.iOutADCArrayCount)
                                {
                                    outletArray_Buffer = new float[par_General.iOutADCArrayCount];
                                    outletArray_Index = 0;
                                }
                                _OutletPress = (lAi[(int)eAI.Outlet] - par_Port.iPxOutAiSubVal) * 100 / par_Port.iPxOutAiDivVal;
                                outletArray_Buffer[outletArray_Index] = (float)Math.Round(_OutletPress + par_Port.dPxOutAiOffsetVal, 1);

                                outletArray_Index = outletArray_Index >= par_General.iOutADCArrayCount - 1 ? 0 : outletArray_Index + 1;
                            }

                            if (par_General.FgSaveOutletADCLog)
                            {
                                string mes = $"ADCLog: Inlet={lAi[(int)eAI.Inlet]} , Outlet={lAi[(int)eAI.Outlet]} , Tempture={_Temperature} , RH={_Humidity}";
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, mes, FoupID);
                            }

                            break;
                        case BoardCommandType.InterlockType:
                            var interlockStatus = dataArray[5] == 0x30 ? true : false;
                            _InterLock = interlockStatus;
                            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"Get board interlock status function={interlockStatus}", FoupID);
                            break;
                        case BoardCommandType.BatchDo:

                            if (dataArray[5] != 0)//error return
                            {
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"Board Batch DO error return.", FoupID);
                                break;
                            }
                            _InterLock = dataArray[6] == 0x30 ? true : false;

                            //DI1 need reverse
                            di1 = dataArray[7] & 0x01;
                            diTemp = dataArray[7];
                            dataArray[7] = (byte)(((diTemp >> 1) << 1) + (di1 > 0 ? 0 : 1));

                            if (_di != dataArray[7] || _do != dataArray[8])
                            {
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"DO Now:{Convert.ToString(dataArray[8], 2).PadLeft(8, '0')} Last:{Convert.ToString(_do, 2).PadLeft(8, '0')}.DI Now:{Convert.ToString(dataArray[7], 2).PadLeft(8, '0')} Last:{Convert.ToString(_di, 2).PadLeft(8, '0')}", FoupID);
                                //add tool record
                                if ((_di & (byte)eDI.CarrierLock) == 0 && ((dataArray[7] & (byte)eDI.CarrierLock) > 0)) //carrier lock 0>>1
                                {
                                    if (par_Port.port_ToolRecord.ClampCounts > (int.MaxValue * 0.95))
                                    {
                                        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Warning, $"ToolRecord Clamp Count Touch Limit,Do reset count.", FoupID);
                                        par_Port.port_ToolRecord.ClampCounts = 0;
                                    }
                                    par_Port.port_ToolRecord.ClampCounts++;
                                }
                                if ((_do & (byte)eDO.Inlet_OFF) == 0 && ((dataArray[8] & (byte)eDO.Inlet_OFF) > 0)) //Inlet_Off 0>>1
                                {
                                    if (par_Port.port_ToolRecord.PurgeCounts > (int.MaxValue * 0.95))
                                    {
                                        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Warning, $"ToolRecord Purge Count Touch Limit,Do reset count.", FoupID);
                                        par_Port.port_ToolRecord.PurgeCounts = 0;
                                    }
                                    par_Port.port_ToolRecord.PurgeCounts++;
                                }


                            }
                            _di = dataArray[7];
                            _do = dataArray[8];

                            break;
                        case BoardCommandType.ErrorDetect:
                            if ((eCommanT_Alarm)dataArray[5] != eCommanT_Alarm.OK)//communication error
                            {
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.BoardError, $"Comunication {((eCommanT_Alarm)dataArray[5]).ToString()}.", FoupID);
                            }
                            if ((eCommanT_Alarm)dataArray[6] != eCommanT_Alarm.OK)//RH error
                            {
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.BoardError, $"RH {((eCommanT_Alarm)dataArray[6]).ToString()}.", FoupID);
                                if (count_RHError == 0)
                                {
                                    time_RHErrorCheck = DateTime.Now;
                                    count_RHError++;
                                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"RHError Count={count_RHError}", FoupID);
                                }
                                else
                                {
                                    if ((DateTime.Now - time_RHErrorCheck).TotalSeconds > par_General.TimeLimitBoardError)
                                    {
                                        time_RHErrorCheck = DateTime.Now;
                                        count_RHError++;
                                        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"RHError Count={count_RHError}", FoupID);
                                    }
                                    if (count_RHError > par_General.CntLimitBoardError && !isAlarm_RHError)
                                    {
                                        isAlarm_RHError = true;
                                        RaiseAlarm(eALID.P1_Board_control_error);
                                        //RaiseAlarm($"L / P {txtPortName.Text} RH sensor detect error.");

                                    }
                                }

                            }
                            else
                            {
                                if (count_RHError > 0)
                                { count_RHError = 0; }
                            }


                            if ((eCommanT_Alarm)dataArray[7] != eCommanT_Alarm.OK)//Temperature error
                            {
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.BoardError, $"Temperature {((eCommanT_Alarm)dataArray[7]).ToString()}.", FoupID);
                            }

                            for (int i = 8; i < dataArray.Length - 2; i++)
                            {
                                string TargetDIO = i < 16 ? "DI" : i < 24 ? "DO" : "AI";
                                if ((eCommanT_Alarm)dataArray[i] != eCommanT_Alarm.OK)
                                {
                                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.BoardError, $"{TargetDIO}{(i % 8) + 1} Abnormal.", FoupID);
                                }
                            }
                            //  if (count_TCommand > par_General.EchoWaitTimeout & !par_General.Fg5DIOWEOFilter && !isAlarm_TCommand)
                            if (count_TCommand > 30 & !par_General.Fg5DIOWEOFilter && !isAlarm_TCommand)
                            {
                                isAlarm_TCommand = true;
                                RaiseAlarm(eALID.P1_Board_control_error);
                                // RaiseAlarm($"L / P {txtPortName.Text} T-LP Main Board Timeout!!");
                            }
                            break;
                    }
                }
                else if (!AlarmEnable)
                {
                    if (!commandReturn.CheckSumResult)
                    {
                        CheckSumErrorCount++;
                        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"CheckSumError Count={CheckSumErrorCount}", FoupID);

                        if (par_General.FgSave5DIOLog)
                        {
                            var dataArray = commandReturn.Bytes.ToArray();
                            var _5DIOLog = BitConverter.ToString(dataArray);
                            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"CheckSumError Receive Data= {_5DIOLog}", FoupID);
                        }

                        if (CheckSumErrorCount > par_General.EchoWaitTimeout)
                        {
                            if (SysStep == eStep.LoadPurge || SysStep == eStep.ContinuePurge || SysStep == eStep.UnloadPurge)
                            {
                                RaiseAlarm(eALID.P1_Board_checksum_error);
                                return;
                            }
                            else
                            {
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"Ignore CheckSumError Alarm When Idle Step.", FoupID);
                                CheckSumErrorCount = 0;
                            }
                        }
                    }
                    else
                    {
                        CheckSumErrorCount = 0;
                    }
                }

                if (receiveDataList.Count > serialPort.ReadBufferSize * 0.5)
                {
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Warning, $"SerialPort_DataReceived data close port read buffer,do clear buffer.", FoupID);
                    receiveDataList.Clear();
                }

            }
            catch (Exception ex)
            { cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"SerialPort_DataReceived({ex.Message})", FoupID); }
        }


        private void ComPortSend(byte[] binaryData)
        {
            try
            {

                lock (sendDataList)
                { sendDataList.Enqueue(binaryData); }

                if (sendDataList.Count % 4 == 0)
                {
                    var _StatusData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.StatusInfo }, _tid);
                    lock (sendDataList)
                    { sendDataList.Enqueue(_StatusData); }
                }

                // serialPort.Write(binaryData, 0, binaryData.Length);
                if (par_General.FgSave5DIOLog)
                {
                    var _5DIOLog = BitConverter.ToString(binaryData);
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Board, $"Send Data: {_5DIOLog}", FoupID);
                }
            }
            catch (ArgumentOutOfRangeException e)
            {
                serialPort.DiscardOutBuffer();
                //  serialPort.DiscardInBuffer();
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"ComPortSend({e.Message})", FoupID);
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"ComPortSend({ex.Message})", FoupID);
            }

        }

        public void SetAirCurtain(bool bEnable)
        {
            try
            {
                if (serialPort.IsOpen)
                {
                    var data = bEnable ? 1 : 0;
                    var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.AirCuratin, (byte)data }, _tid);
                    // serialPort.Write(_binaryData, 0, _binaryData.Length);
                    ComPortSend(_binaryData);
                }
            }
            catch (Exception ex)
            {
                // return false; 
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"SetAirCurtain({ex.Message})", FoupID);
            }
        }

        public byte[] SetInterLock(bool bEnable)
        {
            try
            {
                if (serialPort.IsOpen)
                {
                    var data = bEnable ? 0x30 : 0x31;
                    var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.InterlockType, (byte)data }, _tid);
                    // serialPort.Write(_binaryData, 0, _binaryData.Length);
                    ComPortSend(_binaryData);
                    return _binaryData;
                }
            }
            catch (Exception ex)
            {
                // return false; 
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"SetInterLock({ex.Message})", FoupID);
            }
            return new byte[0];
        }
        public byte[] SetDO_InitialType()
        {
            try
            {
                if (serialPort.IsOpen)
                {
                    byte DO_temp = 0x02;
                    var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.BatchDo, DO_temp }, _tid);
                    // serialPort.Write(_binaryData, 0, _binaryData.Length);
                    ComPortSend(_binaryData);
                    return _binaryData;
                }
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"SetDO({ex.Message})", FoupID);
                // return false; 
            }
            return new byte[0];
        }
        public byte[] SetDOArray(List<eDO> lDIO)
        {
            try
            {
                if (serialPort.IsOpen)
                {
                    UInt16 Sum = 0;
                    for (int i = 0; i < lDIO.Count; i++)
                    { Sum += (byte)lDIO[i]; }
                    //if (bEnable)
                    {
                        // byte DO_temp = (byte)(_do | Sum);
                        byte DO_temp = (byte)Sum;
                        var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.BatchDo, DO_temp }, _tid);
                        // serialPort.Write(_binaryData, 0, _binaryData.Length);
                        ComPortSend(_binaryData);
                        return _binaryData;
                    }
                    //else
                    //{
                    //    byte DO_temp = (byte)(_do & ~Sum);
                    //    var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.BatchDo, DO_temp }, _tid);
                    //    // serialPort.Write(_binaryData, 0, _binaryData.Length);
                    //    ComPortSend(_binaryData);
                    //}

                }
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"SetDO({ex.Message})", FoupID);
                // return false; 
            }
            return new byte[0];
        }
        public byte[] SetDO(byte _dio)
        {
            try
            {
                if (serialPort.IsOpen)
                {
                    byte DO_temp = _dio;
                    var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.BatchDo, DO_temp }, _tid);
                    // serialPort.Write(_binaryData, 0, _binaryData.Length);
                    ComPortSend(_binaryData);
                    return _binaryData;
                }
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"SetDO({ex.Message})", FoupID);
                // return false; 
            }
            return new byte[0];
        }
        public byte[] SetDO(eDO dio, bool bEnable)
        {
            try
            {
                if (serialPort.IsOpen)
                {
                    if (bEnable)
                    {
                        byte DO_temp = (byte)(_do | (byte)dio);
                        var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.BatchDo, DO_temp }, _tid);
                        // serialPort.Write(_binaryData, 0, _binaryData.Length);
                        ComPortSend(_binaryData);
                        return _binaryData;
                    }
                    else
                    {
                        byte DO_temp = (byte)(_do & ~(byte)dio);
                        var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.BatchDo, DO_temp }, _tid);
                        // serialPort.Write(_binaryData, 0, _binaryData.Length);
                        ComPortSend(_binaryData);
                        return _binaryData;
                    }

                }
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"SetDO({ex.Message})", FoupID);
                // return false; 
            }
            return new byte[0];
        }
        public bool GetDI(eDI dio)
        {
            try
            {
                if (par_General.FgPlacementSensorNormalOn && dio == eDI.Placement)
                { return !((_di & (byte)dio) > 0); }
                if (par_General.FgPresenceSensorNormalOn && dio == eDI.Presence)
                { return !((_di & (byte)dio) > 0); }

                if (par_General.FgClampControlFunction)// clamp model carrierlock status base on clamp L&clamp R
                {
                    if (dio == eDI.CarrierLock)
                    {
                        return GetDI(eDI.Clamp_R_ON) && GetDI(eDI.Clamp_L_ON);
                    }
                }
                else
                {
                    if (par_General.FgClampSensorNormalOn && dio == eDI.CarrierLock)
                    { return !((_di & (byte)dio) > 0); }
                }

                return (_di & (byte)dio) > 0;
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"GetDI({ex.Message})", FoupID);
                return false;
            }
        }
        public bool GetDO(eDO dio)
        {
            try
            {
                return (_do & (byte)dio) > 0;
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"GetDO({ex.Message})", FoupID);
                return false;
            }
        }
        public void GetErrorDetect()
        {
            try
            {

                if (serialPort.IsOpen)
                {
                    var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.ErrorDetect }, _tid);
                    // serialPort.Write(_binaryData, 0, _binaryData.Length);
                    ComPortSend(_binaryData);
                }


            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"GetErrorDetect({ex.Message})", FoupID);
                //  return false;
            }
        }
        public void GetBoard_Status()
        {
            try
            {
                if (serialPort.IsOpen)
                {
                    var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.StatusInfo }, _tid);
                    // serialPort.Write(_binaryData, 0, _binaryData.Length);
                    ComPortSend(_binaryData);
                }
                //else
                //{
                //    if (SerialPort.GetPortNames().Contains(serialPort.PortName) && !serialPort.IsOpen)
                //    {
                //        serialPort.Open();
                //        GetBoard_Status();
                //    }
                //}

            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"GetBoard_Status({ex.Message})", FoupID);
                //  return false;
            }

        }
        public void GetBoard_Version()
        {
            try
            {
                if (serialPort.IsOpen)
                {
                    var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.FirmwareCheck }, _tid);
                    // serialPort.Write(_binaryData, 0, _binaryData.Length);
                    ComPortSend(_binaryData);
                }
                //else
                //{
                //    if (SerialPort.GetPortNames().Contains(serialPort.PortName) && !serialPort.IsOpen)
                //    {
                //        serialPort.Open();
                //        GetBoard_Status();
                //    }
                //}

            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"GetBoard_Version({ex.Message})", FoupID);
                //  return false;
            }

        }

        #endregion cmd
        #region autoRun
        public void AutoRun_Start()
        {
            _AutoRunFlag = true;
            Task.Run(() => AutoProcess());
            //AutoProcess();
        }
        public void AutoRun_Stop()
        {
            _AutoRunFlag = false;
            SysStep = SysStep_Temp = eStep.Idle;
            AutoRun_Step = AutoRun_Step_Temp = eProcessStep.None;
        }
        public void AutoRun_Idle()
        {
            if (isPostIdlePurge)//clear post flag
            { isPostIdlePurge = false; }

            currentTimeTick = DateTime.Now;

            SysStep = eStep.Idle;
            AutoRun_Step = eProcessStep.Idle;


            cBrLog.WriteFileLog(par_Port.PortName, cBrLog.eLogType.System, $"Auto Step go to idle step.");
        }
        public void AutoRun_IdlePurge()
        {
            if (!par_General.PurgeEnable)
            {
                cBrLog.WriteFileLog(par_Port.PortName, cBrLog.eLogType.System, $"PurgeEnable=false, Idle Purge Step ByPass.");
                return;
            }
            SysStep = eStep.Idle;
            AutoRun_Step = eProcessStep.Idle_Initial;
            cBrLog.WriteFileLog(par_Port.PortName, cBrLog.eLogType.System, $"Auto Step go to idle purge step.");
        }


        private void CheckSpecialStatus()
        {
            try
            {
                if (!AlarmEnable || (AlarmEnable && AlarmID != eALID.P1_Board_timeout))
                {
                    if (count_MBBoardTimeOut > par_General.EchoWaitTimeout)
                    {
                        RaiseAlarm(eALID.P1_Board_timeout);
                        return;
                    }
                    //check cmd send&recieve time
                    if (Math.Abs(time_MBBoardRecieve - time_MBBoardSend) > 1000 * (count_MBBoardTimeOut + 1))
                    {
                        count_MBBoardTimeOut++;
                        //time_MBBoardSend = time_MBBoardRecieve = Environment.TickCount;
                        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"MB Timeout Count={count_MBBoardTimeOut}.", FoupID);
                    }
                    else if (Math.Abs(time_MBBoardRecieve - time_MBBoardSend) < 1000)
                    {
                        if (count_MBBoardTimeOut != 0)
                        {
                            count_MBBoardTimeOut = 0;
                            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Recover serial port communication ,Reset MB Timeout Count.", FoupID);
                        }
                    }
                }

                //check update port status
                _PortStatus.Name = par_Port.PortName;
                _PortStatus.Warming = isAlarm_TCommand ? "FF" : "00";
                _PortStatus.DI_Status = Convert.ToString(_di, 2).PadLeft(8, '0');
                _PortStatus.DO_Status = Convert.ToString(_do, 2).PadLeft(8, '0');
                _PortStatus.RH = Humidity.ToString();
                _PortStatus.Temp = Temperature.ToString();
                _PortStatus.Flow = Flow.ToString();
                _PortStatus.Inlet = InletPress.ToString();
                _PortStatus.Outlet = OutletPress.ToString();
                _PortStatus.MID = FoupID;

                //check foup in/out
                if (GetDO(eDO.Inlet_ON) == GetDO(eDO.Inlet_OFF))
                {
                    if (!AlarmEnable)
                    {
                        count_InletLikeOutlet++;
                        if (count_InletLikeOutlet > 5)
                        {
                            SetDO_InitialType();
                            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Dectect Inlet and Outlet have same DO status,Do Set Initial DO Status.", FoupID);
                            count_InletLikeOutlet = 0;
                            //isAlarm_BoardTimeout = true;
                            //RaiseAlarm(eALID.PxTLP_MainBoardTimeout);
                        }
                    }
                    // too much log when open com and board not call back.
                    // cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Detect Inlet_ON like Inlet_OFF, Set to Do_InitialType.", FoupID);
                }
                else
                {
                    count_InletLikeOutlet = 0;
                    //check foup in/out
                    bool FoupStatusTemp = false;
                    if (!par_General.FgPlacementByPassModule)//not bypass
                    {
                        if (GetDI(eDI.FoupType) && GetDI(eDI.Placement))
                        {
                            FoupStatusTemp = true;
                            time_FoupInStatusNotMatch = DateTime.Now;
                        }
                        else if (!GetDI(eDI.FoupType) && !GetDI(eDI.Placement))
                        {
                            FoupStatusTemp = false;
                            time_FoupInStatusNotMatch = DateTime.Now;
                        }
                        else
                        {
                            return;
                        }
                    }
                    else//bypass placement
                    {
                        FoupStatusTemp = GetDI(eDI.FoupType);
                        time_FoupInStatusNotMatch = DateTime.Now;
                    }

                    if (FoupInStatus != FoupStatusTemp)
                    {
                        if (((DateTime.Now - time_FoupChangeStart).TotalSeconds > par_General.Int_FoupInOutTimeLimit))
                        {
                            if (FoupStatusTemp)
                            {
                                //send foup in event
                                FoupID = "Unknow ID";
                                //  cSecsManager.Send_S98F11(eCEID.FoupIn, Convert.ToInt32(TID), FoupID);
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Detect Foup In Status Event.", FoupID);

                            }
                            else
                            {
                                //send foup out event
                                FoupID = "";
                                //    cSecsManager.Send_S98F11(eCEID.FoupOut, Convert.ToInt32(TID), FoupID);
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Detect Foup Out Status Event.", FoupID);
                            }
                            FoupInStatus = FoupStatusTemp;
                            time_FoupChangeStart = DateTime.Now;
                        }
                    }
                    else
                    {
                        time_FoupChangeStart = DateTime.Now;
                    }

                }

            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"CheckSpecialStatus({ex.Message})", FoupID);
            }
        }

        private async Task AutoProcess()
        {

            while (_AutoRunFlag)
            {
                try
                {

                    if (serialPort.IsOpen)
                    {

                        if (sendDataList.Count > 0)
                        {
                            lock (sendDataList)
                            { sendData = sendDataList.Dequeue(); }
                            serialPort.Write(sendData, 0, sendData.Length);
                            time_MBBoardSend = Environment.TickCount;
                        }
                        else
                        {
                            GetBoard_Status();
                            lock (sendDataList)
                            { sendData = sendDataList.Dequeue(); }
                            serialPort.Write(sendData, 0, sendData.Length);
                            time_MBBoardSend = Environment.TickCount;
                        }
                    }
                    else
                    {
                        if ((DateTime.Now - time_ManualModeStart).TotalSeconds > time_ManualModeTimeout && !isAutoMode)
                        {
                            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Mode Timeout,Do Change Auto Mode Function.", FoupID);
                            isAutoMode = true;
                        }

                        await Task.Delay(par_General._5DIOScanRate * 100);
                        continue;
                    }



                    { CheckSpecialStatus(); }

                    //if (isAutoMode)
                    {
                        // if (!AlarmEnable || (AlarmEnable && !cSecsManager.ALID_AutoRunNeedStop.Contains(AlarmID)))
                        {
                            switch (SysStep)
                            {
                                case eStep.Idle:
                                    //Flow_Idle_CheckTargetPurge();
                                    Flow_Idle();
                                    break;
                                case eStep.LoadPurge:
                                    //Flow_LoadPurge();
                                    break;
                                case eStep.ContinuePurge:
                                    //Flow_ContinuePurge();
                                    break;
                                case eStep.UnloadPurge:
                                    // Flow_UnLoadPurge();
                                    break;
                                case eStep.PurgeFinish:

                                    break;
                            }

                            if (SysStep != SysStep_Temp)
                            {
                                SysStep_Temp = SysStep;
                                //add step log
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"System step go to {SysStep.ToString()}.", FoupID);
                            }
                            if (AutoRun_Step != AutoRun_Step_Temp)
                            {
                                AutoRun_Step_Temp = AutoRun_Step;
                                //add step log
                                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Auto step go to {AutoRun_Step.ToString()}, System step={SysStep.ToString()}.", FoupID);
                            }
                        }
                    }

                    await Task.Delay(par_General._5DIOScanRate * 100);


                }
                catch (TimeoutException timeoutException)
                {
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"AutoProcess({timeoutException.Message})", FoupID);
                }
                catch (Exception ex)
                {
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"AutoProcess({ex.Message})", FoupID);
                }

            }

        }

        private void Flow_Idle()
        {
            try
            {
                switch (AutoRun_Step)
                {
                    case eProcessStep.Idle:


                        ////move to above check spec and abnormal check
                        //if (TargetPurge_IdlePurge.Count > 0)
                        //{
                        //    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"TargetPurge_IdlePurge Count >0, Do AutoRun_IdlePurge.", FoupID);
                        //    AutoRun_IdlePurge();
                        //    break;
                        //}


                        //if ((DateTime.Now - currentTimeTick).TotalSeconds > par_General.iAbnormalChkStartTim)//check buffer time
                        //{
                        //    // abnormal purge in idle step when flow > 0 and valve not open.
                        //    if (par_General.FgAbnormalPurge && !isAlarm_AbnormalPurge)
                        //    {
                        //        if (!CheckisPurging() && Flow > 0)
                        //        {
                        //            if (Flow < par_Port.iChkLowFlowLimit)
                        //            {
                        //                count_AbnormalPurge = 0;
                        //                //fgAlarm_AbnormalPurge = false;
                        //                //  cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Detect AbnormalPurge End.", FoupID);
                        //            }
                        //            else
                        //            {
                        //                if (count_AbnormalPurge == 0)
                        //                {
                        //                    time_AbnormalPurgeCheck = DateTime.Now;
                        //                    count_AbnormalPurge++;
                        //                    // fgAlarm_AbnormalPurge = true;
                        //                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Detect AbnormalPurge Start.({time_AbnormalPurgeCheck.ToString("HH:mm:ss")},Flow={Flow})", FoupID);
                        //                }

                        //                if (count_AbnormalPurge != 0 && (DateTime.Now - time_AbnormalPurgeCheck).TotalSeconds > par_General.iAbnormalChkStartTim)
                        //                {
                        //                    if (count_AbnormalPurge > par_General.CheckSpecSampleRate)
                        //                    {
                        //                        isAlarm_AbnormalPurge = true;
                        //                        RaiseAlarm(eALID.AbnormalPurgingWhenIdleStep);
                        //                        //RaiseAlarm($"Abnormal Purging when idle step.");
                        //                        SetDO_InitialType();
                        //                    }
                        //                    else
                        //                    {
                        //                        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Detect AbnormalPurge Count={count_AbnormalPurge},Flow={Flow}", FoupID);
                        //                        count_AbnormalPurge++;
                        //                        time_AbnormalPurgeCheck = DateTime.Now;
                        //                    }

                        //                }

                        //            }
                        //        }
                        //        else
                        //        { count_AbnormalPurge = 0; }
                        //    }
                        //    // aircurtain abnormal purge in idle step when flow > 0 and valve not open.
                        //    if (AirCurtainEnable && par_General.FgAirCurtainAbnormalPurgeAlm && !isAlarm_AbnormalPurge_AirCurtain)
                        //    {
                        //        if (!GetDO(eDO.AirCurtain) && AirCurtainFlow > 0)
                        //        {
                        //            if (AirCurtainFlow < par_Port.iChkAirCurtainLowFlowLimit)
                        //            {
                        //                count_AbnormalPurge_AirCurtain = 0;

                        //            }
                        //            else
                        //            {
                        //                if (count_AbnormalPurge_AirCurtain == 0)
                        //                {
                        //                    time_AbnormalPurgeCheck_AirCurtain = DateTime.Now;
                        //                    count_AbnormalPurge_AirCurtain++;
                        //                    // fgAlarm_AbnormalPurge = true;
                        //                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Detect AirCurtain AbnormalPurge Start.({time_AbnormalPurgeCheck_AirCurtain.ToString("HH:mm:ss")},AC Flow={AirCurtainFlow})", FoupID);
                        //                }

                        //                if (count_AbnormalPurge_AirCurtain != 0 && (DateTime.Now - time_AbnormalPurgeCheck_AirCurtain).TotalSeconds > par_General.iAirCurtainAbnormalChkStartTime)
                        //                {
                        //                    if (count_AbnormalPurge_AirCurtain > par_General.iAirCurtainSpecSampleRate)
                        //                    {
                        //                        isAlarm_AbnormalPurge_AirCurtain = true;
                        //                        RaiseAlarm(eALID.AC_AbnormalPurgingWhenIdleStep);
                        //                        //RaiseAlarm($"AirCurtain Abnormal Purging when idle step.");
                        //                        SetDO_InitialType();
                        //                    }
                        //                    else
                        //                    {
                        //                        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Detect AirCurtain AbnormalPurge Count={count_AbnormalPurge_AirCurtain},AC Flow={AirCurtainFlow}", FoupID);
                        //                        count_AbnormalPurge_AirCurtain++;
                        //                        time_AbnormalPurgeCheck_AirCurtain = DateTime.Now;
                        //                    }

                        //                }

                        //            }
                        //        }
                        //        else
                        //        { count_AbnormalPurge_AirCurtain = 0; }
                        //    }
                        //}

                        ////check idle spec
                        //if ((DateTime.Now - time_AutoIdleModeStart).TotalSeconds > par_Port.port_Spec.CheckStartTimeIdle)
                        //{
                        //    //modify check idle spec frequency about 1s
                        //    time_AutoIdleModeStart = DateTime.Now.AddSeconds((par_Port.port_Spec.CheckStartTimeIdle * -1) + 1);

                        //    if (par_Port.port_Spec.FgIdleCheckSpec)
                        //    {
                        //        //RH
                        //        //if (Humidity >= par_Port.port_Spec.iHumiIdle - par_Port.port_Spec.iHumiEXIdle && Humidity <= par_Port.port_Spec.iHumiIdle + par_Port.port_Spec.iHumiEXIdle)
                        //        if (CheckValueInRange((float)Humidity, par_Port.port_Spec.iHumiIdle, par_Port.port_Spec.iHumiEXIdle, "RH"))
                        //        {
                        //            SpecCount_RH = 0;
                        //        }
                        //        else
                        //        {
                        //            SpecCount_RH++;
                        //            if (SpecCount_RH > par_General.CheckSpecSampleRate)
                        //            {
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "RH out of spec on idle step.", FoupID);
                        //                RaiseAlarm(eALID.P1_OutOfSpec_Humidity);
                        //                SpecCount_RH = 0;
                        //                //RaiseAlarm("Detect RH over spec on idle step.");
                        //                break;
                        //            }
                        //        }
                        //        //TEMP
                        //        //  if (Temperature >= par_Port.port_Spec.iTempIdle - par_Port.port_Spec.iTempEXIdle && Temperature <= par_Port.port_Spec.iTempIdle + par_Port.port_Spec.iTempEXIdle)
                        //        if (CheckValueInRange((float)Temperature, par_Port.port_Spec.iTempIdle, par_Port.port_Spec.iTempEXIdle, "Temperature"))
                        //        {
                        //            SpecCount_Temp = 0;
                        //        }
                        //        else
                        //        {
                        //            SpecCount_Temp++;
                        //            if (SpecCount_Temp > par_General.CheckSpecSampleRate)
                        //            {
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "Temperature out of spec on idle step.", FoupID);
                        //                RaiseAlarm(eALID.P1_OutOfSpec_Temperature);
                        //                SpecCount_Temp = 0;
                        //                //RaiseAlarm("Detect Temperature over spec on idle step.");
                        //                break;
                        //            }
                        //        }
                        //        //Flow
                        //        // if (Flow >= par_Port.port_Spec.iFlowIdle - par_Port.port_Spec.iFlowEXIdle && Flow <= par_Port.port_Spec.iFlowIdle + par_Port.port_Spec.iFlowEXIdle)
                        //        if (CheckValueInRange((float)Flow, par_Port.port_Spec.iFlowIdle, par_Port.port_Spec.iFlowEXIdle, "Flow"))
                        //        {
                        //            SpecCount_Flow = 0;
                        //        }
                        //        else
                        //        {
                        //            SpecCount_Flow++;
                        //            if (SpecCount_Flow > par_General.CheckSpecSampleRate)
                        //            {
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "Flow out of spec on idle step.", FoupID);
                        //                if ((float)Flow > par_Port.port_Spec.iFlowIdle)
                        //                {
                        //                    RaiseAlarm(eALID.P1_Flow_High);
                        //                }
                        //                else
                        //                {
                        //                    RaiseAlarm(eALID.P1_Flow_Low);
                        //                }
                        //                SpecCount_Flow = 0;
                        //                //RaiseAlarm("Detect Flow over spec on idle step.");
                        //                break;
                        //            }
                        //        }
                        //        //Inlet
                        //        //if (InletPress >= par_Port.port_Spec.iInletPressureIdle - par_Port.port_Spec.iInletPressureEXIdle && InletPress <= par_Port.port_Spec.iInletPressureIdle + par_Port.port_Spec.iInletPressureEXIdle)
                        //        if (CheckValueInRange((float)InletPress, par_Port.port_Spec.iInletPressureIdle, par_Port.port_Spec.iInletPressureEXIdle, "Inlet"))
                        //        {
                        //            SpecCount_Inlet = 0;
                        //        }
                        //        else
                        //        {
                        //            SpecCount_Inlet++;
                        //            if (SpecCount_Inlet > par_General.CheckSpecSampleRate)
                        //            {
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "Inlet out of spec on idle step.", FoupID);
                        //                RaiseAlarm(eALID.P1_OutOfSpec_Inlet);
                        //                SpecCount_Inlet = 0;
                        //                //RaiseAlarm("Detect InletPress over spec on idle step.");
                        //                break;
                        //            }
                        //        }
                        //        //Outlet
                        //        //if (OutletPress >= par_Port.port_Spec.iOutletPressureIdle - par_Port.port_Spec.iOutletPressureEXIdle && OutletPress <= par_Port.port_Spec.iOutletPressureIdle + par_Port.port_Spec.iOutletPressureEXIdle)
                        //        if (CheckValueInRange((float)OutletPress, par_Port.port_Spec.iOutletPressureIdle, par_Port.port_Spec.iOutletPressureEXIdle, "Outlet"))
                        //        {
                        //            SpecCount_Outlet = 0;
                        //        }
                        //        else
                        //        {
                        //            SpecCount_Outlet++;
                        //            if (SpecCount_Outlet > par_General.CheckSpecSampleRate)
                        //            {
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "Outlet out of spec on idle step.", FoupID);
                        //                RaiseAlarm(eALID.P1_OutOfSpec_Outlet);
                        //                SpecCount_Outlet = 0;
                        //                //  RaiseAlarm("Detect OutletPress over spec on idle step.");
                        //                break;
                        //            }
                        //        }
                        //    }

                        //    if (AirCurtainEnable && par_Port.port_Spec.FgAirCurtainIdleCheckSpec)
                        //    {
                        //        //AirCurtain_Flow
                        //        // if (AirCurtainFlow >= par_Port.port_Spec.iSpecAirCurtainFlowIdle - par_Port.port_Spec.iSpecAirCurtainFlowIdleEx && AirCurtainFlow <= par_Port.port_Spec.iSpecAirCurtainFlowIdle + par_Port.port_Spec.iSpecAirCurtainFlowIdleEx)
                        //        if (CheckValueInRange((float)AirCurtainFlow, par_Port.port_Spec.iSpecAirCurtainFlowIdle, par_Port.port_Spec.iSpecAirCurtainFlowIdleEx, "AC_Flow"))
                        //        {
                        //            SpecCount_Aircurtain_Flow = 0;
                        //        }
                        //        else
                        //        {
                        //            SpecCount_Aircurtain_Flow++;
                        //            if (SpecCount_Aircurtain_Flow > par_General.iAirCurtainSpecSampleRate)
                        //            {
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "Air curtain Flow out of spec on idle step.", FoupID);
                        //                if ((float)AirCurtainFlow > par_Port.port_Spec.iSpecAirCurtainFlowIdle)
                        //                {
                        //                    RaiseAlarm(eALID.P1_AC_Flow_High);
                        //                }
                        //                else
                        //                {
                        //                    RaiseAlarm(eALID.P1_AC_Flow_Low);
                        //                }
                        //                SpecCount_Aircurtain_Flow = 0;
                        //                //RaiseAlarm("Detect AirCutrain Flow over spec on idle step.");
                        //                break;
                        //            }
                        //        }
                        //        //AirCurtain_Inlet
                        //        //if (AirCurtainInletPress >= par_Port.port_Spec.iSpecAirCurtainInletPressureIdle - par_Port.port_Spec.iSpecAirCurtainInletPressureIdleEx && AirCurtainInletPress <= par_Port.port_Spec.iSpecAirCurtainInletPressureIdle + par_Port.port_Spec.iSpecAirCurtainInletPressureIdleEx)
                        //        if (CheckValueInRange((float)AirCurtainInletPress, par_Port.port_Spec.iSpecAirCurtainInletPressureIdle, par_Port.port_Spec.iSpecAirCurtainInletPressureIdleEx, "AC_Inlet"))
                        //        {
                        //            SpecCount_Aircurtain_Inlet = 0;
                        //        }
                        //        else
                        //        {
                        //            SpecCount_Aircurtain_Inlet++;
                        //            if (SpecCount_Aircurtain_Inlet > par_General.iAirCurtainSpecSampleRate)
                        //            {
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "Air curtain Inlet out of spec on idle step.", FoupID);
                        //                RaiseAlarm(eALID.P1_AC_OutOfSpec_Inlet);
                        //                SpecCount_Aircurtain_Inlet = 0;
                        //                //RaiseAlarm("Detect AirCutrain InletPress over spec on idle step.");
                        //                break;
                        //            }

                        //        }
                        //    }

                        //}



                        break;
                    case eProcessStep.Idle_Initial:

                        //if (!par_General.FgPlacementByPassModule)
                        //{
                        //    //if (FoupInStatus)
                        //    if (GetDI(eDI.FoupType) || GetDI(eDI.Placement))
                        //    {

                        //        Flow_Idle_resetCount(true);
                        //        //count_IdlePurgeRetry_SetInterlock = 0;
                        //        //count_IdlePurgeRetry = count_IdlePurgeRetry_Outlet = count_IdlePurgeRetry_Aircurtain = 0;
                        //        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Idle_Initial fial, (FoupType ,Placement =>{GetDI(eDI.FoupType)} ,{GetDI(eDI.Placement)}).", FoupID);
                        //        //time_IdlePurge = DateTime.Now;
                        //        //time_IdlePurge_Aircurtain = DateTime.Now;
                        //        TargetPurge_IdlePurge.Clear();
                        //        AutoRun_Step = eProcessStep.Idle_PurgeEnd;
                        //        break;
                        //    }
                        //}
                        //else
                        //{
                        //    if (GetDI(eDI.FoupType))
                        //    {
                        //        Flow_Idle_resetCount(true);
                        //        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Idle_Initial fial, (FoupType ,Placement =>{GetDI(eDI.FoupType)} ,ByPass).", FoupID);
                        //        TargetPurge_IdlePurge.Clear();
                        //        AutoRun_Step = eProcessStep.Idle_PurgeEnd;
                        //        break;
                        //    }
                        //}

                        //if (InterLock)
                        //{
                        //    CurrentCMDCheck = SetInterLock(false);
                        //    currentTimeTick = DateTime.Now;
                        //}
                        //count_IdlePurgeRetry_SetInterlock = 0;


                        //AutoRun_Step = eProcessStep.Idle_PurgeStart;

                        break;
                    case eProcessStep.Idle_PurgeStart:
                        //if (FoupInStatus)
                        //{
                        //    Flow_Idle_resetCount(true);
                        //    //count_IdlePurgeRetry_SetInterlock = 0;
                        //    //count_IdlePurgeRetry = count_IdlePurgeRetry_Outlet = count_IdlePurgeRetry_Aircurtain = 0;
                        //    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Foup In when Idle PurgeStart.Go to Idle_PurgeEnd step", FoupID);
                        //    //time_IdlePurge = DateTime.Now;
                        //    //time_IdlePurge_Aircurtain = DateTime.Now;
                        //    AutoRun_Step = eProcessStep.Idle_PurgeEnd;
                        //    break;
                        //}

                        //if (!InterLock)
                        //{
                        //    ////avoid intlet on & off have same type.
                        //    //if (!TargetPurge_IdlePurge.Contains((eDO)0x01) && !TargetPurge_IdlePurge.Contains((eDO)0x02))
                        //    //{ TargetPurge_IdlePurge.Add(eDO.Inlet_OFF); }

                        //    CurrentCMDCheck = SetDOArray(TargetPurge_IdlePurge);
                        //    time_PurgeStart = DateTime.Now;
                        //    currentTimeTick = DateTime.Now;
                        //    if (TargetPurge_IdlePurge.Contains(eDO.Inlet_ON) && !GetDO(eDO.Inlet_ON))
                        //    {
                        //        if (!isLeakIdlePurge)
                        //        {
                        //            time_IdlePurge = DateTime.Now;
                        //        }
                        //        else
                        //        {
                        //            time_LeakStart = DateTime.Now;
                        //            time_LeakWait = DateTime.Now;//for initial with start time
                        //        }
                        //    }
                        //    if (TargetPurge_IdlePurge.Contains(eDO.AirCurtain) && !GetDO(eDO.AirCurtain))
                        //    {
                        //        time_IdlePurge_Aircurtain = DateTime.Now;
                        //    }

                        //    AutoRun_Step = eProcessStep.Idle_Purging;
                        //}
                        //else
                        //{
                        //    if (count_IdlePurgeRetry_SetInterlock > par_General.EchoWaitTimeout)
                        //    {
                        //        isAlarm_IdlePurge_SetInterlock = true;
                        //        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "Set Interlock not successful on idle step.", FoupID);
                        //        RaiseAlarm(eALID.PxTLP_MainBoardTimeout);
                        //        //  RaiseAlarm($"SetInterlock Fail When Idle_PurgeStart Step.");
                        //        break;
                        //    }

                        //    if (!sendDataList.Contains(CurrentCMDCheck) && (DateTime.Now - currentTimeTick).TotalSeconds > currentTimeTickWaitTime)
                        //    {
                        //        CurrentCMDCheck = SetInterLock(false);
                        //        currentTimeTick = DateTime.Now;
                        //        count_IdlePurgeRetry_SetInterlock++;
                        //    }
                        //}
                        break;
                    case eProcessStep.Idle_Purging:
                        //// if (FoupInStatus)
                        //if (GetDI(eDI.FoupType))
                        //{
                        //    if (par_General.FgPlacementByPassModule)
                        //    {
                        //        Flow_Idle_resetCount(true);
                        //        //count_IdlePurgeRetry_SetInterlock = 0;
                        //        //count_IdlePurgeRetry = count_IdlePurgeRetry_Outlet = count_IdlePurgeRetry_Aircurtain = 0;
                        //        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Foup In when Idle Purging.Go to Idle_PurgeEnd step", FoupID);
                        //        //time_IdlePurge = DateTime.Now;
                        //        //time_IdlePurge_Aircurtain = DateTime.Now;
                        //        AutoRun_Step = eProcessStep.Idle_PurgeEnd;
                        //        break;
                        //    }
                        //    else if (!par_General.FgPlacementByPassModule && GetDI(eDI.Placement))
                        //    {
                        //        Flow_Idle_resetCount(true);
                        //        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Foup In when Idle Purging.Go to Idle_PurgeEnd step", FoupID);
                        //        AutoRun_Step = eProcessStep.Idle_PurgeEnd;
                        //        break;
                        //    }
                        //}

                        ////check status and add count after send command 
                        //if (sendDataList.Contains(CurrentCMDCheck) || (DateTime.Now - currentTimeTick).TotalSeconds < currentTimeTickWaitTime)
                        //{
                        //    break;
                        //}

                        //bool isPurgeRetry = false;
                        //foreach (eDO dO in TargetPurge_IdlePurge)
                        //{
                        //    switch (dO)
                        //    {
                        //        case eDO.Inlet_ON:
                        //            if (!GetDO(dO))
                        //            {
                        //                count_IdlePurgeRetry++;
                        //                if (!isLeakIdlePurge)
                        //                {
                        //                    // SetDOArray(TargetPurge_IdlePurge);
                        //                    time_IdlePurge = DateTime.Now;
                        //                }
                        //                else
                        //                {
                        //                    time_LeakCheck = DateTime.Now;
                        //                }
                        //                isPurgeRetry = true;
                        //            }
                        //            break;
                        //        case eDO.Outlet:
                        //            if (!GetDO(dO))
                        //            {
                        //                //SetDOArray(TargetPurge_IdlePurge);
                        //                count_IdlePurgeRetry_Outlet++;
                        //                time_IdlePurge = DateTime.Now;
                        //                isPurgeRetry = true;
                        //            }
                        //            break;
                        //        case eDO.AirCurtain:
                        //            if (!GetDO(dO))
                        //            {
                        //                //SetDOArray(TargetPurge_IdlePurge);
                        //                count_IdlePurgeRetry_Aircurtain++;
                        //                time_IdlePurge_Aircurtain = DateTime.Now;
                        //                isPurgeRetry = true;
                        //            }
                        //            break;
                        //    }

                        //    if (par_General.FgRetryTurnOnValveAlm)
                        //    {
                        //        if (count_IdlePurgeRetry > par_General.IdlePurgeControlFailNum || count_IdlePurgeRetry_Outlet > par_General.IdlePurgeControlFailNum)
                        //        {
                        //            isAlarm_IdlePurgeRetry = true;
                        //            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "Idle purging open valve retry fail.", FoupID);
                        //            RaiseAlarm(eALID.Purging_ValveTurnONFail);
                        //            //RaiseAlarm($"Purge Retry Timeout When Idle_Purging Step.");
                        //        }
                        //        if (count_IdlePurgeRetry_Aircurtain > par_General.AirCurtainIdlePurgeControlFailNum)
                        //        {
                        //            isAlarm_IdlePurgeRetry = true;
                        //            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, " AirCurtain idle purging open valve retry fail.", FoupID);
                        //            RaiseAlarm(eALID.AC_Purging_ValveTurnONFail);
                        //            //  RaiseAlarm($"AirCurtain Purge Retry Timeout When Idle_Purging Step.");
                        //        }
                        //    }

                        //    if (isPurgeRetry)
                        //    { break; }
                        //}
                        //if (isPurgeRetry)
                        //{
                        //    AutoRun_Step = eProcessStep.Idle_PurgeStart;
                        //    break;
                        //}


                        //if (TargetPurge_IdlePurge.Count == 0)
                        //{
                        //    if (!InterLock)
                        //    {
                        //        //count_IdlePurgeRetry_SetInterlock = 0;
                        //        //count_IdlePurgeRetry = count_IdlePurgeRetry_Outlet = count_IdlePurgeRetry_Aircurtain = 0;
                        //        CurrentCMDCheck = SetInterLock(true);
                        //    }

                        //    currentTimeTick = DateTime.Now;
                        //    Flow_Idle_resetCount(false);//not reset idle purge time,because reset on check purge time finish

                        //    if (!isLeakIdlePurge)//change step when not leak.leak purge change purge end after wait time and get inlet.
                        //    {
                        //        AutoRun_Step = eProcessStep.Idle_PurgeEnd;
                        //        break;
                        //    }
                        //}

                        //if (!isPostIdlePurge && !isLeakIdlePurge) // idle purge
                        //{
                        //    if ((DateTime.Now - time_IdlePurge).TotalSeconds > par_General.PxIdlePurgeTime)//check purge time
                        //    {
                        //        if (GetDO(eDO.Inlet_ON) || GetDO(eDO.Outlet) || !GetDO(eDO.Inlet_OFF))//turn off purge when purge time done.
                        //        {
                        //            var doTemp = _do;
                        //            byte temp = (byte)(((byte)(((doTemp >> 2) << 2) + eDO.Inlet_OFF)) & ~(byte)eDO.Outlet);
                        //            SetDO(temp);
                        //            //SetDO_InletValve(false);  
                        //            //SetDO(eDO.Outlet);
                        //            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"IdlePurgeTime Full Time, Turn Off Purge.", FoupID);
                        //            time_IdlePurge = DateTime.Now;//resset purge end time
                        //            currentTimeTick = DateTime.Now;
                        //            if (TargetPurge_IdlePurge.Contains(eDO.Inlet_ON))
                        //            {
                        //                TargetPurge_IdlePurge.Remove(eDO.Inlet_ON);
                        //                // time_IdlePurge = DateTime.Now;
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"TargetPurge_IdlePurge remove Inlet_ON.", FoupID);
                        //            }
                        //            if (TargetPurge_IdlePurge.Contains(eDO.Outlet))
                        //            {
                        //                TargetPurge_IdlePurge.Remove(eDO.Outlet);
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"TargetPurge_IdlePurge remove Outlet.", FoupID);
                        //            }

                        //        }
                        //    }

                        //    if ((DateTime.Now - time_IdlePurge_Aircurtain).TotalSeconds > par_General.PxAirCurtainIdlePurgeTime && GetDO(eDO.AirCurtain))//check aircurtain purge time
                        //    {
                        //        if (TargetPurge_IdlePurge.Contains(eDO.Inlet_ON))
                        //        {
                        //            SetDO(eDO.AirCurtain, false);
                        //        }
                        //        else
                        //        {
                        //            SetDO_InitialType();
                        //        }
                        //        //SetDO(eDO.AirCurtain, false);
                        //        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Aircurtain IdlePurgeTime Full Time, Turn Off Aircurtain Purge.", FoupID);
                        //        time_IdlePurge_Aircurtain = DateTime.Now;//resset aircurtain purge end time
                        //        currentTimeTick = DateTime.Now;
                        //        if (TargetPurge_IdlePurge.Contains(eDO.AirCurtain))
                        //        {
                        //            TargetPurge_IdlePurge.Remove(eDO.AirCurtain);
                        //            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"TargetPurge_IdlePurge Remove AirCurtain.", FoupID);
                        //        }
                        //        //if (TargetPurge_IdlePurge.Contains(eDO.Inlet_OFF))
                        //        //{
                        //        //    TargetPurge_IdlePurge.Remove(eDO.Inlet_OFF);
                        //        //}

                        //    }
                        //}
                        //else if (isPostIdlePurge)// post idle purge
                        //{
                        //    if ((DateTime.Now - time_IdlePurge).TotalSeconds > par_General.iPost_PurgeTimeSetting)//check post purge time
                        //    {
                        //        if (GetDO(eDO.Inlet_ON) || GetDO(eDO.Outlet) || !GetDO(eDO.Inlet_OFF) || GetDO(eDO.AirCurtain))//turn off purge when post purge time done.
                        //        {
                        //            SetDO_InitialType();
                        //            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Post IdlePurgeTime Full Time, Go to InitialType.", FoupID);
                        //            time_IdlePurge = DateTime.Now;//resset purge end time
                        //            time_IdlePurge_Aircurtain = DateTime.Now;
                        //            currentTimeTick = DateTime.Now;
                        //            if (TargetPurge_IdlePurge.Contains(eDO.Inlet_ON))
                        //            {
                        //                TargetPurge_IdlePurge.Remove(eDO.Inlet_ON);
                        //                // time_IdlePurge = DateTime.Now;
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Post TargetPurge_IdlePurge remove Inlet_ON.", FoupID);
                        //            }
                        //            if (TargetPurge_IdlePurge.Contains(eDO.Outlet))
                        //            {
                        //                TargetPurge_IdlePurge.Remove(eDO.Outlet);
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Post TargetPurge_IdlePurge remove Outlet.", FoupID);
                        //            }
                        //            if (TargetPurge_IdlePurge.Contains(eDO.AirCurtain))
                        //            {
                        //                TargetPurge_IdlePurge.Remove(eDO.AirCurtain);
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Post TargetPurge_IdlePurge Remove AirCurtain.", FoupID);
                        //            }

                        //        }
                        //    }
                        //    //else if ((DateTime.Now - time_IdlePurge_Aircurtain).TotalSeconds > par_General.iPost_PurgeTimeSetting && GetDO(eDO.AirCurtain))//check aircurtain post purge time
                        //    //{
                        //    //    SetDO(eDO.AirCurtain, false);
                        //    //    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Aircurtain IdlePurgeTime Full Time, Turn Off Aircurtain Purge.", FoupID);
                        //    //    time_IdlePurge_Aircurtain = DateTime.Now;//resset aircurtain purge end time
                        //    //    currentTimeTick = DateTime.Now;
                        //    //    if (TargetPurge_IdlePurge.Contains(eDO.AirCurtain))
                        //    //    {
                        //    //        TargetPurge_IdlePurge.Remove(eDO.AirCurtain);
                        //    //        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"TargetPurge_IdlePurge Remove AirCurtain.", FoupID);
                        //    //    }

                        //    //}
                        //}
                        //else if (isLeakIdlePurge) //leak idle purge
                        //{
                        //    if ((DateTime.Now - time_LeakStart).TotalSeconds > par_General.Int_CheckLeakPurgeTimeLimit)//check leak purge time
                        //    {
                        //        if (GetDO(eDO.Inlet_ON) || !GetDO(eDO.Inlet_OFF))//turn off purge when leak purge time done.
                        //        {
                        //            SetDO_InitialType();
                        //            currentTimeTick = DateTime.Now;
                        //            if (TargetPurge_IdlePurge.Contains(eDO.Inlet_ON))
                        //            {
                        //                TargetPurge_IdlePurge.Remove(eDO.Inlet_ON);
                        //                // time_IdlePurge = DateTime.Now;
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Leak TargetPurge_IdlePurge remove Inlet_ON.", FoupID);
                        //            }
                        //            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Leak IdlePurgeTime Full Time,Do Close valve and wait keeptime.", FoupID);
                        //        }
                        //        else
                        //        {
                        //            if (TargetPurge_IdlePurge.Count == 0)
                        //            {

                        //                LeakInletStart = InletPress;
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Record Leak Start Inlet={LeakInletStart}.", FoupID);
                        //                time_LeakWait = DateTime.Now;
                        //                AutoRun_Step = eProcessStep.Idle_PurgeEnd;
                        //                break;
                        //            }
                        //        }
                        //    }

                        //}

                        ////only aircurtain idle purge will exist inlet off target do.
                        //if (TargetPurge_IdlePurge.Count == 1 && TargetPurge_IdlePurge.Contains(eDO.Inlet_OFF))
                        //{
                        //    TargetPurge_IdlePurge.Remove(eDO.Inlet_OFF);
                        //}


                        break;
                    case eProcessStep.Idle_PurgeEnd:

                        //if ((DateTime.Now - currentTimeTick).TotalSeconds < currentTimeTickWaitTime)
                        //{ break; }
                        //if (_do != (byte)eDO.Inlet_OFF)//initial type
                        //{
                        //    SetDO_InitialType();
                        //    currentTimeTick = DateTime.Now;
                        //    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Idle_PurgeEnd Set DO_InitialType .", FoupID);
                        //    count_IdlePurgeRetry++;
                        //    if (count_IdlePurgeRetry > par_General.IdlePurgeControlFailNum)
                        //    {
                        //        isAlarm_IdlePurgeRetry = true;
                        //        if (GetDO(eDO.AirCurtain))
                        //        {
                        //            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "AirCurtain idle purging close valve retry fail.", FoupID);
                        //            RaiseAlarm(eALID.AC_ValveTurnOffFail);
                        //        }
                        //        if (GetDO(eDO.Inlet_ON) || !GetDO(eDO.Inlet_OFF))
                        //        {
                        //            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "Idle purging close valve retry fail.", FoupID);
                        //            RaiseAlarm(eALID.ValveTurnOffFail);
                        //        }

                        //        //RaiseAlarm($"Purge Stop Retry Timeout When Idle_PurgeEnd Step.");
                        //    }
                        //    if (!InterLock)
                        //    {
                        //        count_IdlePurgeRetry_SetInterlock = 0;
                        //        CurrentCMDCheck = SetInterLock(true);
                        //    }
                        //}
                        //else if (!InterLock)
                        //{
                        //    if (count_IdlePurgeRetry_SetInterlock > par_General.EchoWaitTimeout)
                        //    {
                        //        isAlarm_IdlePurge_SetInterlock = true;
                        //        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, "Set Interlock not successful on idle step.", FoupID);
                        //        RaiseAlarm(eALID.PxTLP_MainBoardTimeout);
                        //        //RaiseAlarm($"SetInterlock Fail When Idle_PurgeEnd Step.");
                        //    }

                        //    if (!sendDataList.Contains(CurrentCMDCheck))
                        //    {
                        //        currentTimeTick = DateTime.Now;
                        //        count_IdlePurgeRetry_SetInterlock++;
                        //        CurrentCMDCheck = SetInterLock(true);
                        //    }
                        //}
                        //else
                        //{
                        //    if (!isLeakIdlePurge)//idle and post purge
                        //    {
                        //        AutoRun_Step = eProcessStep.PurgeFinish;
                        //    }
                        //    else
                        //    {
                        //        if (FoupInStatus)//stop leak when foup in
                        //        {
                        //            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Give up Leak IdlePurge Wait Time When Foup In.", FoupID);
                        //            AutoRun_Step = eProcessStep.PurgeFinish;
                        //        }
                        //        else if ((DateTime.Now - time_LeakWait).TotalSeconds > par_General.Long_CheckLeakWaitLimit)//check leak purge time
                        //        {

                        //            LeakInletEnd = InletPress;
                        //            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Record Leak End Inlet={LeakInletEnd}.", FoupID);
                        //            //  double PressProportion = Math.Abs((LeakInletStart - LeakInletEnd) / LeakInletStart);
                        //            double PressProportion = (LeakInletStart - LeakInletEnd) / LeakInletStart;
                        //            PressProportion = Math.Round(PressProportion, 4);
                        //            if (PressProportion > par_General.Dbl_CheckLeakIntPressProportion)
                        //            {
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Leak Check Fail.(Now={PressProportion} Tartget={par_General.Dbl_CheckLeakIntPressProportion})", FoupID);
                        //                RaiseAlarm(eALID.P1_LeakCheckFail);
                        //            }
                        //            else
                        //            {
                        //                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Leak Check OK.", FoupID);
                        //            }
                        //            time_LeakCheck = DateTime.Now;//leak purge end reset time
                        //            AutoRun_Step = eProcessStep.PurgeFinish;
                        //        }
                        //    }

                        //}

                        break;
                    case eProcessStep.PurgeFinish:

                        //if (isPostIdlePurge)//clear post flag
                        //{ isPostIdlePurge = false; }

                        //if (isLeakIdlePurge)//clear leak flag
                        //{ isLeakIdlePurge = false; }

                        ////clear purge count and do not reset purge time at idle step
                        //Flow_Idle_resetCount(false);

                        //if (TargetPurge_IdlePurge.Count > 0)
                        //{ TargetPurge_IdlePurge.Clear(); }

                        currentTimeTick = DateTime.Now;
                        AutoRun_Step = eProcessStep.Idle;
                        break;
                    case eProcessStep.None:
                        AutoRun_Step = eProcessStep.Idle;
                        break;
                }
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"Flow_Idle({ex.Message})", FoupID);
            }

        }

        #endregion autoRun


        //private bool CheckValueInRange(float _dataValue, float _centerValue, float _deltaValue, string ItemName)
        //{
        //    //_centerValue=0 mean bypass
        //    if (_centerValue == 0)
        //    { return true; }

        //    if (_dataValue <= _centerValue + _deltaValue && _dataValue >= _centerValue - _deltaValue)
        //    { return true; }
        //    else
        //    {
        //        cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Procession, $"Detect {ItemName} over spec range.Current={_dataValue},range=({_centerValue - _deltaValue}~{_centerValue + _deltaValue}).", FoupID);
        //        return false;
        //    }
        //}

        private bool CheckisPurging()
        {
            try
            {
                if (!par_General.FgTwoStepFlowFunction)
                {
                    if (GetDO(eDO.Inlet_ON) && !GetDO(eDO.Inlet_OFF))
                    {
                        return true;
                    }
                    else
                    {
                        return false;
                    }

                }
                else
                {
                    if (GetDO(eDO.Inlet_H) && !GetDO(eDO.Inlet_OFF))
                    {
                        return true;
                    }
                    else if (GetDO(eDO.Inlet_L) && !GetDO(eDO.Inlet_OFF))
                    {
                        return true;

                    }
                    else
                    { return false; }
                }
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"CheckisPurging({ex.Message})", FoupID);
                return false;
            }
        }

        private void SetDO_InletValve(bool bOpen)
        {
            try
            {
                if (serialPort.IsOpen)
                {
                    var Do_temp = _do & (byte)252;
                    if (bOpen)
                    {
                        Do_temp = Do_temp + (int)eDO.Inlet_ON;
                        var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.BatchDo, (byte)Do_temp }, _tid);
                        // serialPort.Write(_binaryData, 0, _binaryData.Length);
                        ComPortSend(_binaryData);

                        //btnValve_H.BackColor = Color.MistyRose;

                    }
                    else
                    {
                        Do_temp = Do_temp + (int)eDO.Inlet_OFF;
                        var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.BatchDo, (byte)Do_temp }, _tid);
                        // serialPort.Write(_binaryData, 0, _binaryData.Length);
                        ComPortSend(_binaryData);

                        //btnValve_H.BackColor = Color.LightGray;

                    }

                }
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"SetDO_InletValve({ex.Message})", FoupID);
                // return false; 
            }
        }

        private void SetDO_InletValve(bool bOpen, bool isInlet_H)//for two flow function
        {
            try
            {
                if (serialPort.IsOpen)
                {
                    var Do_temp = _do & (byte)244;
                    if (bOpen)
                    {
                        if (isInlet_H)
                        { Do_temp = Do_temp + (int)eDO.Inlet_H; }
                        else
                        { Do_temp = Do_temp + (int)eDO.Inlet_L; }
                        var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.BatchDo, (byte)Do_temp }, _tid);
                        // serialPort.Write(_binaryData, 0, _binaryData.Length);
                        ComPortSend(_binaryData);

                        //btnValve_H.BackColor = Color.MistyRose;

                    }
                    else
                    {
                        Do_temp = Do_temp + (int)eDO.Inlet_OFF;
                        var _binaryData = board_util.combineData(board_util.BR_COMMAND_SEND_START_BIT, new byte[] { (byte)BoardCommandType.BatchDo, (byte)Do_temp }, _tid);
                        // serialPort.Write(_binaryData, 0, _binaryData.Length);
                        ComPortSend(_binaryData);

                        //btnValve_H.BackColor = Color.LightGray;

                    }

                }
            }
            catch (Exception ex)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.Exception, $"SetDO_InletValve({ex.Message})", FoupID);
                // return false; 
            }
        }

        private void btnValve_H_Click(object sender, EventArgs e)
        {
            if (isAutoMode || DisableCOM)
            { return; }
            if (!par_General.PurgeEnable)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Reject Purge Because PurgeEnable=false.", FoupID);
                return;
            }

            if (GetDO(eDO.Inlet_ON) && !GetDO(eDO.Inlet_OFF))
            {
                SetDO_InletValve(false);
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Turn Off Inlet Valve.", FoupID);
            }
            else if (!GetDO(eDO.Inlet_ON) && GetDO(eDO.Inlet_OFF))
            {
                SetDO_InletValve(true);
                time_PurgeStart = DateTime.Now;
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Turn On Inlet Valve.", FoupID);
            }
            else
            {
                SetDO_InletValve(false);
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Turn Off Inlet Valve.", FoupID);
            }

        }

        private void btnValve_L_Click(object sender, EventArgs e)
        {
            if (isAutoMode || DisableCOM)
            { return; }
            if (!par_General.PurgeEnable)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Reject Purge Because PurgeEnable=false.", FoupID);
                return;
            }
            //SetDO(eDO.Inlet_L, !GetDO(eDO.Inlet_L));
            if (GetDO(eDO.Inlet_L) && !GetDO(eDO.Inlet_OFF))
            {
                SetDO_InletValve(false, false);
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Turn Off Inlet Valve.", FoupID);
            }
            else if (!GetDO(eDO.Inlet_L) && GetDO(eDO.Inlet_OFF))
            {
                SetDO_InletValve(true, false);
                time_PurgeStart = DateTime.Now;
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Turn On Inlet_L Valve.", FoupID);
            }
            else
            {
                SetDO_InletValve(false, false);
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Turn Off Inlet Valve.", FoupID);
            }
        }

        public void btnValve_AirCurtain_Click(object sender, EventArgs e)
        {
            if (isAutoMode || DisableCOM)
            { return; }
            //safe funciton
            if (AirCurtainEnable)
            {
                if (GetDO(eDO.AirCurtain))
                {
                    SetDO(eDO.AirCurtain, false);
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Turn Off AirCurtain Valve.", FoupID);
                }
                else
                {
                    SetDO(eDO.AirCurtain, true);
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Turn On AirCurtain Valve.", FoupID);
                }
            }


        }

        private void picOutValve_Click(object sender, EventArgs e)
        {
            if (isAutoMode || DisableCOM)
            { return; }
            if (!par_General.PurgeEnable)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Reject Purge Because PurgeEnable=false.", FoupID);
                return;
            }
            if (GetDO(eDO.Outlet))
            {
                SetDO(eDO.Outlet, false);
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Turn Off Outlet Valve.", FoupID);
            }
            else
            {
                SetDO(eDO.Outlet, true);
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Turn On Outlet Valve.", FoupID);
            }
            //SetDO(eDO.Outlet, !GetDO(eDO.Outlet));
        }

        public void btnPurge_Click(object sender, EventArgs e)
        {
            if (isAutoMode || DisableCOM)
            { return; }
            if (!par_General.PurgeEnable)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Reject Purge Because PurgeEnable=false.", FoupID);
                return;
            }

            if (InterLock && !GetDI(eDI.CarrierLock))
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Reject Purge Because InterLock=On and carrierLock=false.", FoupID);
                return;
            }

            Button btn = (Button)sender;
            if (btn == btnPurge)
            {
                if (btnPurge.BackColor == Color.Lime && GetDO(eDO.Inlet_OFF))
                {
                    SetDOArray(new List<eDO> { eDO.Inlet_ON, eDO.Outlet });
                    time_PurgeStart = DateTime.Now;
                    btnPurge.BackColor = Color.RoyalBlue;
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Purge Turn On.({UserID} Click)", FoupID);

                }
                else if (btnPurge.BackColor == Color.RoyalBlue && !GetDO(eDO.Inlet_OFF))
                {
                    SetDO_InitialType();
                    btnPurge.BackColor = Color.Lime;
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Purge Turn Off.({UserID} Click)", FoupID);

                }


            }
            else if (btn == btnEndPurgeCycle)
            {
                if (btnEndPurgeCycle.BackColor == Color.Yellow && GetDO(eDO.Inlet_OFF))
                {
                    SetDOArray(new List<eDO> { eDO.Inlet_ON, eDO.Outlet });
                    time_PurgeStart = DateTime.Now;
                    btnEndPurgeCycle.BackColor = Color.RoyalBlue;
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"EndPurgeCycle Turn On.({UserID} Click)", FoupID);

                }
                else if (btnEndPurgeCycle.BackColor == Color.RoyalBlue && !GetDO(eDO.Inlet_OFF))
                {
                    SetDO_InitialType();
                    btnEndPurgeCycle.BackColor = Color.Yellow;
                    cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"EndPurgeCycle Turn Off.({UserID} Click)", FoupID);

                }

            }

        }

        public void btnGiveUpPurging_Click(object sender, EventArgs e)
        {

        }

        private void btnEndPurgeCycle_Click(object sender, EventArgs e)
        {
            if (isAutoMode || DisableCOM)
            { return; }
            if (!par_General.PurgeEnable)
            {
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Reject Purge Because PurgeEnable=false.", FoupID);
                return;
            }
            //todo: need check function
            cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"End Purge Cycle Start.({UserID} Click)", FoupID);
        }

        private void btnClamp_Click(object sender, EventArgs e)
        {
            if (isAutoMode || DisableCOM)
            { return; }
            if (GetDO(eDO.Clamp))
            {
                SetDO(eDO.Clamp, false);
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Control Clamp Turn Off.", FoupID);
            }
            else
            {
                SetDO(eDO.Clamp, true);
                cBrLog.WriteFileLog(txtPortName.Text, cBrLog.eLogType.System, $"Manual Control Clamp Turn On.", FoupID);
            }

        }
    }

    public class ucTLP_Status
    {
        public string Name { get; set; }
        public string Warming { get; set; }
        public string DI_Status { get; set; }
        public string DO_Status { get; set; }
        public string RH { get; set; }
        public string Temp { get; set; }
        public string Flow { get; set; }
        public string Inlet { get; set; }
        public string Outlet { get; set; }
        public string MID { get; set; }
    }

    namespace BrCommand.Board.Util
    {
        public enum BoardCommandType
        {
            Unknow = 0,

            FirmwareCheck = 0x31,

            StatusInfo = 0x32,
            StatusInfo_UTS3in1 = 0x36,
            StausInfo_MFC = 0x4d,

            BatchDo = 0x4f,
            SingleDo = 0x53,
            SingleDoOneShot = 0x73,
            ErrorDetect = 0x54,
            InterlockType = 0x55,

            SensorType_DI = 0x74,
            SensorType_RH_MFC = 0x43,

            DisplayLCD = 0x64,
            RFIDRead = 0x52,
            AirCuratin = 0x61,
        }
        public static class board_util
        {
            static readonly byte[] BR_COMMAND_CHECK_LIST = { 0x30, 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x38, 0x39, 0x41, 0x42, 0x43, 0x44, 0x45, 0x46 };
            public static readonly byte BR_COMMAND_SEND_START_BIT = 0x02;
            public static readonly byte BR_COMMAND_READ_START_BIT = 0x03;
            public static readonly byte BR_COMMAND_LEN_START_BIT = 0x30;
            public static byte[] genCheckSum(byte[] data)
            {
                byte[] result = { 0x00, 0x00 };
                int sum = 0;
                foreach (byte item in data)
                    sum += item;

                result[0] = BR_COMMAND_CHECK_LIST[(sum / 16) % 16];
                result[1] = BR_COMMAND_CHECK_LIST[sum % 16];
                return result;
            }

            public static bool checkData(byte[] data, byte[] checkSum)
            {
                bool result = false;
                byte[] sum = genCheckSum(data);
                result = sum.SequenceEqual(checkSum);
                return result;
            }

            public static byte[] combineData(byte beginByte, byte[] data, string tid)
            {
                List<byte> tmp = new List<byte>();
                tmp.Add(beginByte);
                tmp.Add((byte)(data.Length + 0x02 + BR_COMMAND_LEN_START_BIT));
                byte[] _tid = tid.Substring(0, 2).Select(item => (byte)item).ToArray();
                tmp.AddRange(_tid);
                tmp.AddRange(data);
                tmp.AddRange(genCheckSum(_tid.Concat(data).ToArray()));
                return tmp.ToArray();
            }

            public static byte[] fromBigEndian(this byte[] data)
            {
                if (BitConverter.IsLittleEndian)
                {
                    return data.Reverse().ToArray();
                }
                return data;
            }

            public static byte[] fromLittleEndian(this byte[] data)
            {
                if (!BitConverter.IsLittleEndian)
                {
                    return data.Reverse().ToArray();
                }
                return data;
            }

            public static byte[] toBigEndian(this byte[] data)
            {
                return data.fromBigEndian();
            }

            public static byte[] toLittleEndian(this byte[] data)
            {
                return data.fromLittleEndian();
            }
        }

        public static class BoardUtil
        {
            public static List<byte> SplitBoardCommand(ref List<byte> data)
            {
                List<byte> result = new List<byte>();

                //BR cmd rule receive data need to be 03 and data length > 100 or length <=0
                while ((data.Count > 0 && data[0] != 0x03) || (data.Count > 1 && (data[1] - 0x30) > 100) || (data.Count > 1 && (data[1] - 0x30) <= 0))
                {
                    data.RemoveAt(0);
                }

                if (data.Count > 1)
                {
                    int length = data[1] - 0x30;

                    //if (length < 0)
                    //{
                    //    data.Clear();
                    //    return result;
                    //}

                    if (length + 4 <= data.Count)
                    {
                        result.AddRange(data.Take(length + 4));
                        data.RemoveRange(0, length + 4);

                        if (!board_util.checkData(result.Skip(2).Take(length).ToArray(), result.Skip(length + 2).Take(2).ToArray()))
                        {
                            result.Clear();
                        }

                    }
                }

                return result;
            }
            public static BoardResult SplitBrCommand(ref List<byte> data)
            {
                BoardResult board = new BoardResult();
                board.Bytes = new List<byte>();
                board.CheckSumResult = true;
                //  var result = board.Bytes;

                //BR cmd rule receive data need to be 03 and data length > 100 or length <=0
                while ((data.Count > 0 && data[0] != 0x03) || (data.Count > 1 && (data[1] - 0x30) > 100) || (data.Count > 1 && (data[1] - 0x30) <= 0))
                {
                    data.RemoveAt(0);
                }

                if (data.Count > 1)
                {
                    int length = data[1] - 0x30;

                    //if (length < 0)
                    //{
                    //    data.Clear();
                    //    return result;
                    //}

                    if (length + 4 <= data.Count)
                    {
                        board.Bytes.AddRange(data.Take(length + 4));
                        data.RemoveRange(0, length + 4);

                        if (!board_util.checkData(board.Bytes.Skip(2).Take(length).ToArray(), board.Bytes.Skip(length + 2).Take(2).ToArray()))
                        {
                            //board.Bytes.Clear();
                            board.CheckSumResult = false;
                        }

                    }
                }

                return board;
            }
            public class BoardResult
            {
                public List<byte> Bytes { get; set; } = new List<byte>();
                public bool CheckSumResult { get; set; } = true;
            }
        }
    }
}
