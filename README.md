# EG-1004-Cardier-Card-Dispenser-
  #Rad Project Workfile
  import time
  import cv2
  import RPi.GPIO as GPIO
  import time
  from picamera2 import Picamera2
  import random
  
  # facial detection algorithm
  faceCascade = cv2.CascadeClassifier("/home/eg1004/Downloads/haarfile.xml")
  
  # GPIO stuff
  GPIO.setwarnings(False)
  GPIO.setmode(GPIO.BCM)
  
  # stepper motor
  control_pins = [5, 6, 16, 26]
  # Control pin Duty Cycle
  for pin in control_pins:
      GPIO.setup(pin, GPIO.OUT)
      GPIO.output(pin, 0)
  
  halfstep_seq = [
      [1, 0, 0, 0],
      [1, 1, 0, 0],
      [0, 1, 0, 0],
      [0, 1, 1, 0],
      [0, 0, 1, 0],
      [0, 0, 1, 1],
      [0, 0, 0, 1],
      [1, 0, 0, 1]]
  
  # servo
  # duty cycle, calibrate if needed
  MIN_DUTY = 5
  MAX_DUTY = 10
  
  servo_signal_pin = 12
  GPIO.setup(servo_signal_pin, GPIO.OUT)
  
  servo = GPIO.PWM(servo_signal_pin, 50)
  servo.start(0)
  
  def deg_to_duty(deg):
      return (deg - 0) * (MAX_DUTY - MIN_DUTY) / 180 + MIN_DUTY
  
  
  # small DC motor (select)
  motor_in1 = 17  # Connect to IN1 on the L298N
  motor_in2 = 18  # Connect to IN2 on the L298N
  enable_pin = 25  # Connect to ENA on the L298N
  
  motor_in3 = 22
  motor_in4 = 23
  enable_pin2 = 24
  
  GPIO.setup(motor_in1, GPIO.OUT)
  GPIO.setup(motor_in2, GPIO.OUT)
  GPIO.setup(enable_pin, GPIO.OUT)
  
  GPIO.setup(motor_in3, GPIO.OUT)
  GPIO.setup(motor_in4, GPIO.OUT)
  GPIO.setup(enable_pin2, GPIO.OUT)
  
  pwm = GPIO.PWM(enable_pin, 100)  # 100 Hz frequency
  pwm2 = GPIO.PWM(enable_pin2, 100)  # 100 Hz frequency
  
  pwm.start(0)
  pwm2.start(0)
  
  # big DC motor (launch)
  big_motor = 21
  GPIO.setup(big_motor, GPIO.OUT)
  pwm3 = GPIO.PWM(big_motor, 500)
  pwm3.start(0)
  
  # other initializations
  
  waitTime = 50  # milliseconds
  counter = 0  # number of faces detected
  confidence_level = 3
  
  # cap = cv2.VideoCapture(0)
  # time.sleep(0.5)  # let camera warm up
  
  rotations = 500  # have to figure out a good number for this
  
  resolution_x = 1280
  resolution_y = 960
  
  camera = Picamera2()
  camera.start()
  
  #buttons are equal to a gpio pin implement later on for test
  button_1 = 0
  button_2 = 0
  
  #Rotates Base
  def rotate_base(rot):
      for i in range(rot):
          for halfstep in range(8):
              for pin in range(4):
                  GPIO.output(control_pins[pin], halfstep_seq[halfstep][pin])
              time.sleep(0.001)  # change for speed
  
  #Checks confidence 
  def check_confidence(faces_detected, conf_lvl):
      matches = []
      for i in range(len(faces_detected)):
          counter = 1
          for j in range(i + 1, len(faces_detected)):
              param_match = 0
              for k in range(3):
                  if k < 2:
                      if abs(faces_detected[i][k] - faces_detected[j][k]) / 2 < 20:
                          param_match += 1
                  else:
                      w, h = standardize_rect(faces_detected[i][k], faces_detected[i][k + 1])
                      w2, h2 = standardize_rect(faces_detected[j][k], faces_detected[j][k + 1])
                      if (abs(w - w2) / 2) < 20 and \
                              (abs(h - h2) / 2) < 20:
                          param_match += 2
              if param_match == 4:
                  counter += 1
          if counter == conf_lvl:
              matches.append(faces_detected[i])
  
      return matches
  
  def detect_face(output=None):
      # ret, img = cap.read()  # capture image from video
  
      # if cv2.waitKey(5) & 0xFF == ord('b'):  # does not work right now
      #     cap.release()
      #     cv2.destroyAllWindows()
      img = camera.capture_array()
  
  
      if cv2.waitKey(waitTime):  # after a certain amount of time, detect for a new face
          faces = faceCascade.detectMultiScale(img, scaleFactor=1.05, minNeighbors=5, minSize=(150, 150))
          for (x, y, w, h) in faces:
              faces_detected.append((x, y, w, h))
              # print(x, y, w, h)
  
      if isinstance(output, list):
          for i in output:
              cv2.rectangle(img, (i[0], i[1]), (i[0] + i[2], i[1] + i[3]), (255, 0, 0), 4)
              print("Face detected")
  
          cv2.imshow("Output", img)
  
      #camera.close()
  
  #Makes Rectangle for PI Camera
  def standardize_rect(w, h):
          scale_factor = w / 100
          return [int(w / scale_factor), int(h / scale_factor)]
  
  def manual_onecard():
      if_pressed = False
      while if_pressed == False:
          if_pressed = GPIO.output(button_1, GPIO.HIGH)
      select()
      launch()
  
  def select():
      GPIO.output(motor_in1, GPIO.HIGH)
      GPIO.output(motor_in2, GPIO.LOW)
          
      GPIO.output(motor_in3, GPIO.HIGH)
      GPIO.output(motor_in4, GPIO.LOW)
  
      var = random.randint(1,2)
      if var == 1:
          pwm.ChangeDutyCycle(25)  # 50% duty cycle
      else:
          pwm2.ChangeDutyCycle(25)  # 50% duty cycle
  
  def launch(power):
      pwm3.ChangeDutyCycle(power)
  
  def first_round(num_cards):
      rotations = 50 # CHANGE THIS NUMBER ONCE WE HAVE ACTUAL DATA
      count_rotations = 0
      count_face = 0
      while count_rotations < rotations:
          detected = False
          
          faces_detected = []
  
          rotate_base(rotations)
          count_rotations += 1
  
          for i in range(confidence_level):
              detect_face()
  
          match = check_confidence(faces_detected, confidence_level)
          print(match)
  
          if len(match) == 0:
              print("No face detected")
          else:
              detected = True
              count_face += 1
  
          faces_detected = []
          detect_face(match)
  
          if detected:
              print("Face detected")
              for i in range(num_cards):
                  select()
                  launch()
              
  
      
  def main():
      c_g = 0
      c_nc = 0
      timer = 0 
      timer_2 = 0
      count = 0
      button_2 = GPIO.input(24)
      button_1 = GPIO.input(26)
      led = GPIO.input(15)
     
      while True:
          if GPIO.input(25): #Number of pin can be changed I just put 25 there for now 
              button_gamemode_pressed = False
          while button_gamemode_pressed == False:
              if led ==  GPIO.input(15):
                  GPIO.output(led,GPIO.HIGH) #LED light on ready for input 
              button_gamemode_pressed = GPIO.output(button_2, GPIO.HIGH)
          button_gamemode_pressed = False
          timer.start()
          GPIO.output(led,GPIO.LOW) #lED OFF
          c_g += 1
          GPIO.output(button_2,GPIO.LOW)
          while timer < 5:
              if GPIO.output(button_2, GPIO.HIGH):
                  c_g += 1
                  timer = 0
                  timer.start()
                  GPIO.output(button_2, GPIO.LOW)
  
          if c_g == 1: #manual select starts here
              timer_2.start()
              
              while timer_2 < 5:
                  if GPIO.output(button_2, GPIO.HIGH):
                      c_nc += 1
                      timer_2 = 0
                      timer_2.start()
                      GPIO.output(button_2,GPIO.LOW)
  
              first_round(c_nc)
                      
          elif c_g == 2:
              first_round(2)
  
              timer = 0
              bj_count = 0
              rotations = 50 # CHANGE THIS NUMBER ONCE WE HAVE ACTUAL DATA
              count_rotations = 0
              count_face = 0
              while count_rotations < rotations:
                  detected = False
                  
                  faces_detected = []
  
                  rotate_base(rotations)
                  count_rotations += 1
  
                  for i in range(confidence_level):
                          if detected == True:
                              detect_face()
  
                  match = check_confidence(faces_detected, confidence_level)
                  print(match)
  
                  if len(match) == 0:
                      print("No face detected")
                  else:
                      detected = True
                      count_face += 1
  
                  faces_detected = []
                  detect_face(match)
  
                  if detected:
                      print("Face detected")
                      stand = False
                      while stand == False:
                          timer.start()
                          
                          while timer < 5:
                              if GPIO.output(button_1, GPIO.HIGH):
                                  bj_count +=1
                              if bj_count == 1:  #Stand 
                                  
                                  rotate_base()
                                  print("Stand")
                              else:  
                                  manual_onecard()
                                  print("hit")
                                  # add function for manual_dispense()
                      
                  #Poker starts here 
          elif c_g == 3:
              #Poker flop and dispense
              first_round(2)
              for i in range(3):
                  select()
                  launch()
                  while True:# Flop
                      while  GPIO.output(button_2,GPIO.LOW):
                          if GPIO.output(button_2,GPIO.HIGH):
                              manual_onecard()
                              counter +=1
                      while counter > 2:
                          manual_onecard() #tURN
  
                          
                  timer = 0
  
              if time < 5:
                  GPIO.output(button_2,GPIO.LOW) 
                  c_g += 1
  
  
          elif c_g == 4:
              while timer < 5:
                  timer.stop()
                  c_nc = 5
                  launch(5)
  
  
          elif c_g == 5:
              manual_onecard()
  
                
  
  
  
  
          
          
  
  
