The hardware configuration is relatively straightforward, comprising the following fundamental components:
•	ATMEGA 328p Microcontroller on an Arduino Uno
•	2x High tourque servo motors
•	A Cheap Webcam
•	A Laser (duh)
•	An old Laptop
I joined the two servos, one for pan, the other for tilt, and mounted the laser to that assembly. Then the signal wires of the servos are connected to the Arduino, and all of the components are fed 5v. Of course, some simple instructions need to be written onto the Arduino
        #include
        
        Servo serX;
        Servo serY;
        
        String tempModify;
        
        void setup() {
            serX.attach(11);
            serY.attach(10);
            Serial.begin(9600);
            Serial.setTimeout(10);
        }
        
        void loop() {
            //lol
        }
        
        void serialEvent() {
            tempModify = Serial.readString();
            serX.write(parseDataX(tempModify));
            serY.write(parseDataY(tempModify));
        }

        int parseDataX(String data){
            data.remove(data.indexOf(":"));
            data.remove(data.indexOf("X"), 1);
            return data.toInt();
        }
        
        int parseDataY(String data){
            data.remove(0,data.indexOf(":") + 1);
            data.remove(data.indexOf("Y"), 1);
            return data.toInt();
        }
                         
With these instructions, the microcontroller listens for a serial event, processes the serial data, and writes it to the servos. The structure I've devised for positioning data is as follows:
X(X position):Y(Y Position)
This data is going to be constantly sent via serial communication from my laptop, updating the servo's position many times a second.
The Important Inner Workings
Now lets get into how the heart of the system work, the facial recognition and positional logic. For my facial recognition framework, I used Emgu CV (which is just a .Net wrapper for Open CV). For my GUI, I used WinForms, because I have previous experience with it and it has existing integration with Emgu CV.
After getting Emgu CV installed onto my project (easier said than done) I added 2 ImageBox controls to my form. ImageBox is a custom Emgu control and is similar to WinForms' PictureBox, but optimized for CV applications. I'll set these up to give me a before and after view of the facial recognition process.
 
The next step is to set up a Capture object that allows us to get input from the webcam. This is simply done by instantiating a new Capture object and calling QueryFrame on it. This will return a single frame from the camera, we can display it by setting the Image property of the ImageBox to the returned image. Of course we need to do this many times a second for steady video. My solution was to set up a Timer and add the method where we perform a frame capture as an event handler to the Timer's Tick event, which occurs every time the Timer reaches a given interval. Therefore, we process a new frame every 33 milliseconds (in the given implementation)

        private void initializeTimer()
        {
            timer.Tick += new EventHandler(processFrameAndUpdateImg);
            timer.Interval = 33;
            timer.Start();
        }
        Now that we're getting a steady stream of images, let's perform facial recognition on them. Let's abstract away the logic to perform this in a separate class named ImageRecognition.
        public static class ImageRecognition
        {
            private static List<PointF> facePositions = new List<PointF>();
            private static GpuCascadeClassifier classifier  = new GpuCascadeClassifier(@"haarcascade_frontalface_default.xml");
        
            public static Image<Bgr, byte> detectFace(Image<Bgr, byte> image, out List<PointF> positions)
            {
                Image<Bgr, byte> copyImage = new Image<Bgr, byte>(image.Bitmap); 
                facePositions.Clear();
        
                using (GpuImage<Bgr, Byte> gpuImage = new GpuImage<Bgr, byte>(image))
                using (GpuImage<Gray, Byte> gpuGray = gpuImage.Convert<Gray, Byte>())  
                {  
                    foreach (var face in classifier.DetectMultiScale(gpuGray, 1.2, 10, Size.Empty))
                    {
                        copyImage.Draw(face, new Bgr(Color.Red), 4);
                        facePositions.Add(new PointF(face.Location.X + (face.Width / 2), face.Location.Y + (face.Height / 2)));
                    }
                    positions = facePositions;
                }
                return copyImage;
            }
        }


There are quite a few things happening here, so I'll try and break it down. There are 2 class member variables to avoid repeat instantiation since we are going to call detectFace many times a second. We're loading up a classifier that comes standard with Emgu CV, though it's very possible to use another or even train your own, this is a good, general purpose frontal face classifier. The detectFace method takes in an image and we immediately copy it to a new image object. We do this because Image is a mutable type, so any changes here will be reflected elsewhere (in this case, when we add it to the ImageBox). Then, with a couple of using statements to prevent garbage collection from freaking out, we convert the image to a GpuImage of type BGR, then we convert that to a GpuImage of type Gray, because the classifier we are using was trained in grayscale. Next we iterate over a collection of the detected faces returned by the call:
classifier.DetectMultiScale(gpuGray, 1.2, 10, Size.Empty)
This method takes in our image, a scale factor argument for window scaling between subsequent scans (default 1.2), a minimum neighbors argument which is used to base object neighbor grouping on (default is 4, but I find a more strict grouping for a general classifier like ours is beneficial), and finally minimum window size, setting this to Size.Empty uses the default size of the samples the classifier was trained on.


        foreach (var face in classifier.DetectMultiScale(gpuGray, 1.2, 10, Size.Empty))
        {
            copyImage.Draw(face, new Bgr(Color.Red), 4);
            facePositions.Add(new PointF(face.Location.X + (face.Width / 2), face.Location.Y + (face.Height / 2)));
        }


We want this implementation supports multiple face detection, which is why we iterate over the Rectangle Array returned by DetectMultiScale. On each iteration, a red rectangle is drawn onto the image and its position is added to a list. All that's left to do is set the output list to the positions list and return the image.
Okay cool, so now we just take the coordinates from the output list, send them to the servos and we're done right...
Nope.
That's where I assumed it would end as well, but upon testing, I found that the system was fairly accurate when I was in in the center of its field of view, but wildly overshot when I was near FOV's bounds. It took me way too long to figure out what was going on, but from there it was a relatively straightforward fix.
The issue was the the webcam has about a 50° viewing angle, while the servo configuration has 180° of angular range. So applying data taken from the webcam to the servos would cause a scaling factor as inputs reached the outer bounds of the webcam's FOV, ie me moving to the sides of the room.
  
 
To fix the issue, we need to scale servo instructions to the bounds of the webcam's degrees of vision. This is ultimately accomplished through calibration (which I will touch on in a moment) and in the following method:


        private Point calculatePoint(PointF point)
        {
            return (new Point(((int)(settings.xRightCalibration - (point.X / xScaleConst))), ((int)(settings.yTopCalibration - (point.Y / yScaleConst)))));
        } 
        But as you can see, some data must be collected before we can execute this line. With lines this complex, I always find it easier to think about what each part does to achieve the end goal, so lets break it down.
        private void calculateDivisionConstants()
        {
            xScaleConst = (captureFrameWidth) / (settings.xRightCalibration - settings.xLeftCalibration);
            yScaleConst = (captureFrameHeight) / (settings.yTopCalibration - settings.yBotCalibration);
        }


Here we can see how xScaleConst and yScaleConst are calculated. So captureFrameWidth and captureFrameHeight are fairly self explanatory, the values can be attained by calling Width and Height on the Image returned by our frame capture. The rest of the values are properties of settings, but what is the settings object? Settings is the product of the calibration I mentioned earlier. After realizing that FOV to angular range scaling was an issue I had to create an entire calibration system to deal with it.
 
 
Aside from being a terrible piece of UI, the calibration system allows the user to manually control the laser. The idea is to bring the laser to each bound of the screen (ie the webcam's FOV limits), and click save once it is there. This is what the variables xRightCalibration, xLeftCalibration, yTopCalibration and yBotCalibration represent. This data is then serialized to JSON format and saved to disk, which means you don't need to calibrate it every time you restart the system. This calibration system helps to account for variability's in the robot's location. For example, if I were to hard code the values and move the robot, it could loose accuracy due to changes in orientation relative to the wall.
After that's complete, it's a simple matter of deserializing the stored JSON and instantiating a new generic from it, in our case, named settings. So the calibration properties simply indicate where the laser meets the bounds of the webcam's FOV. We can use these values along with the image's proportions to calculate the xScaleConst and yScaleConst scaling values.


        xScaleConst = (captureFrameWidth) / (settings.xRightCalibration - settings.xLeftCalibration);
        yScaleConst = (captureFrameHeight) / (settings.yTopCalibration - settings.yBotCalibration);
        private Point calculatePoint(PointF point)
        {
            return (new Point(((int)(settings.xRightCalibration - (point.X / xScaleConst))), ((int)(settings.yTopCalibration - (point.Y / yScaleConst)))));
        }


Now that we've got our scaling values, we can apply them to the points capture by facial recognition. This is done in the (point.X / xScaleConst) and (point.Y / yScaleConst) of the above line. Finally, I subtract the right most bound and top most bound of the screen by these respective values. This is because the axes of my servo configuration are inverted, so I must invert my inputs as well.


        private void sendSignal(Point currentPoint)
        {
            serialPort.Write("X" + (currentPoint.X + settings.xShift).ToString() + ":Y" + (currentPoint.Y + settings.yShift).ToString());
        }


Finally, we can write this data to a SerialPort control, and output it to the microcontroller, which will point the servos at our face.
Of course, there are loads of things I left out of this article that were in my final build, including: hardware angle of freedom limitations, processing multi-face detection, voice control, voice response, ease of use settings, coordinate tweaking for lasering the eye, and much more. But this is a good overview of the major components of the system, I hope you enjoyed reading.
