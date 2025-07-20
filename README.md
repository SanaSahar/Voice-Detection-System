# Voice-Detection-System
%Signals and Systems End Semester Project 
%-----------------GUI BASED VOICE DETECTOR----------------------------------
function Voice_Detector       % main function


%------------------------------Creates the GUI window---------------------------

    fig = figure('Name', 'VOICE DETECTOR', 'NumberTitle', 'off','Position', [200 80 650 550]); %lbwh

    uicontrol('Style', 'pushbutton', 'String', 'Record Sound', 'FontSize', 12, 'Position', [50 450 200 40], 'Callback', @recordSound);
 
    uicontrol('Style', 'pushbutton', 'String', 'Play Sound', 'FontSize', 12, 'Position', [400 450 200 40], 'Callback', @playSound);

    uicontrol('Style', 'pushbutton', 'String', 'Analyze', 'FontSize', 12, 'Position', [225 400 200 40], 'Callback', @analyze); %LBWH

   
%--------------------------Axes for Plots------------------------------------

    ax1 = axes('Units', 'pixels', 'Position', [50 230 250 150]);
    ax2 = axes('Units', 'pixels', 'Position', [350 230 250 150]);
    %-------------Text Box to Display the Result (type, energy, pitch)----------

    resultBox = uicontrol('Style', 'text', 'FontSize', 11, 'HorizontalAlignment', 'left', 'Position', [50 50 550 150], 'String', 'Result will display here.');

    % ------------Store GUI data for future use-------------
    data.fs = 44100;           % Sampling frequency 
    data.audio = [];
    guidata(fig, data);

    
    
% --------------------- RECORD SOUND FUNCTION ---------------------
    
    function recordSound(~, ~) %function input argument nhi use kr rha 
        data = guidata(fig);
        recorder = audiorecorder(data.fs, 16, 1);    % 16-bit mono recorder (built in)%memory allocate
        msgbox('Recording 5 seconds... Speak now!');
        recordblocking(recorder, 5);  %sec
        data.audio = getaudiodata(recorder);  
        guidata(fig, data);
        msgbox('Recording Complete!');
    end


% --------------------- PLAY SOUND FUNCTION ---------------------
    function playSound(~, ~)
        data = guidata(fig);
        if isempty(data.audio)
            errordlg('Please record a sound first.');
            return;
        end
        sound(data.audio, data.fs);
    end

% --------------------- ANALYZE FUNCTION ---------------------
    function analyze(~, ~)
        data = guidata(fig);
        if isempty(data.audio)
            errordlg('No sound recorded.');
            return;
        end

        fs = data.fs;
        audio = data.audio / max(abs(data.audio));  % Normalize make 
        len = length(audio);
        t = linspace(0, len/fs, len);

        % Noise Reduction using Moving Average
        window = 100;
        noise_red = ones(1, window) / window;
        smooth = conv(audio, noise_red, 'same'); %length same 

        axes(ax1);
        plot(t, audio, 'r', t, smooth, 'b');
        legend('Original', 'Smoothed');
        title('Noise Reduction');
        xlabel('Time (s)');
        ylabel('Amplitude');

        % Extract 1-second frame from middle
        mid = floor(len/2 - fs/2);
        frame = smooth(mid:mid + fs - 1);

        
%--------------- Energy and Power------------
t_frame = linspace(0, 1, length(frame));  % Time vector for 1-second frame
energy = trapz(t_frame, abs(frame).^2);
duration = 1;  % 1-second
power = energy / duration;



        % detect pitch freq
        [r, lags] = xcorr(frame, fs/2, 'coeff');
        r = r(lags >= 20);
        lags = lags(lags >= 20);
        [~, lag] = max(r);
        pitch_freq = fs / lags(lag);

        % Zero Crossing Rate
        zc = sum(abs(diff(frame > 0)));

        % Classification using Pitch first
        if pitch_freq >= 80 && pitch_freq <= 180
            type = 'Male Voice';
        elseif pitch_freq > 180 && pitch_freq <= 300
            type = 'Female Voice';
        elseif pitch_freq > 300 && pitch_freq <= 700
            type = 'Child Voice';
        elseif pitch_freq > 700 && pitch_freq <= 1200
            type = 'Bird Sound';
        elseif pitch_freq > 1200 && pitch_freq <= 5000
            type = 'Cat or Animal Sound';
        elseif pitch_freq > 5000
            type = 'Other/Unknown Sound';
        else
            type = 'Unclassified';
        end
        
        
        
    % Signal Type Classification
    if isfinite(energy) && duration > 0
        signalType = 'Finite Duration Signal , It is an Energy Signal';
        powerInfo = 'Power is theoretically Infinite/Undetermined';
    else
        signalType = 'Infinite Duration Signal ,Power Signal';
        powerInfo = '';
    end

        % Zero crossing only applied if pitch is unclassified
        if strcmp(type, 'Unclassified') && zc > 400
            type = 'Animal Sound';
        end

        % FFT Plot
        f = linspace(0, fs/2, floor(length(frame)/2));
        Y = abs(fft(frame));
        Y = Y(1:length(f));

        axes(ax2);
        plot(f, Y);
        title('Frequency Domain');
        xlabel('Frequency (Hz)');
        ylabel('Magnitude');
% Display Results
    result1 = sprintf('Detected: %s', type);
    result2 = sprintf('Pitch Frequency: %.2f Hz', pitch_freq);
    result3 = sprintf('Energy: %.5f', energy);
    result4 = sprintf('%s', signalType);

    fullResult = sprintf('%s\n%s\n%s\n%s\n%s\n\n%s', result1, result2, result3, result4, powerInfo);

    set(resultBox, 'String', fullResult);
end

end

