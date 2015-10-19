// moves around randomly with light, notices when follower gets too close
// 1 distance sensor, 2 LEDs

#include </home/craig/Downloads/libpololu-avr/pololu/3pi.h>
#include <pololu/3pi.h>

// 3pi buzzer sounds
const char welcome[] PROGMEM = ">>c32>>f32";
const char go[] PROGMEM = "L16 cdegreg4";
const char lost[] PROGMEM = "f8<e8";
const char backup[] PROGMEM = "L16 >>dr>>dr>>d";
const char ledoff[] PROGMEM = "L32 ara";
const char ledon[] PROGMEM = "L32 >er>e";
const char followed[] PROGMEM = ">>f16>e16>>f16>e16>>f16>e16";
// declare variables
int start_dist, dist, start_time, time;
int l_turn, f_move;
int left_wheel, right_wheel;
int max_speed = 80;
unsigned int edge_sensors[5];
int distthresh = 500;
int min_dist = 1500;
int edgethresh = 500;
int edge_calib[5];
int time;

int i;

void wait_for_buttonB() {
	// wait for button press, play welcome sound
	while(!button_is_pressed(BUTTON_B) && !button_is_pressed(BUTTON_C)) { // wait for button B to be pressed 
		delay_ms(10);
	}
	play_from_program_space(welcome);
	if(button_is_pressed(BUTTON_C)) {
		button_c_loop(); // used for testing and demonstrations
	}
	while(button_is_pressed(BUTTON_B)) { // wait for button B to be released
		delay_ms(10);
	}
}

void setup() {
	// play music
	pololu_3pi_init(2000); // needed for using sensors, sets timeout value
	set_digital_output(IO_D1, LOW);  // make trigger pin low
	set_digital_output(IO_C5, HIGH); // turn guidance LEDs on
	play_from_program_space(welcome);
	wait_for_buttonB();
}

int arraymax(unsigned int array[]) { // find max element in array
	unsigned int max = 0;
	int i;
	//int size = 5; (for edge detectors)
	int size = sizeof(array) / sizeof(array[0]);
	for(i = 0; i < size; i++) {
		if(array[i] > max) {
			max = array[i];
		}
	}
	return max;
}

int button_c_loop() { // used for testing and demonstrations
	// don't move, just freak out if too close to distance sensor
	while(1) {
		set_motors(0, 0); // the robot shouldn't ever move forward
		time = get_time_from_sensor();
		if(time < min_dist) {// double-check
			time = get_time_from_sensor();
		}

		if(time < min_dist) { // follower too close //(time < start_time - distthresh)
			play_from_program_space(followed);
			set_digital_output(IO_C5, LOW);
			
			set_motors(0, 0);
			delay_ms(500);
			set_motors(100, -100);
			delay_ms(300);
			set_motors(0, 0);
			delay_ms(100);
			set_motors(-100, 100);
			delay_ms(600);
			set_motors(0, 0);
			delay_ms(100);
			set_motors(100, -100);
			delay_ms(300);
			set_motors(0, 0);
			delay_ms(100);
			
			delay_ms(100);
			for(i = 0; i < 10; i++) {
				set_motors(100, 100);
				delay_ms(30);
				set_motors(-100, -100);
				delay_ms(30);
			}
			set_motors(0, 0);
			
			set_digital_output(IO_C5, HIGH);
			play_from_program_space(ledon);
			delay_ms(1000);
			continue;
		}
		stop_playing();
		set_digital_output(IO_C5, HIGH);
		set_motors(0, 0);
	}
}

int get_time_from_sensor() {
	set_digital_output(IO_D1, HIGH);  // make a 10 us pulse on TRIG
	delay_us(10);
	set_digital_output(IO_D1, LOW);
	while (!is_digital_input_high(IO_D0));  // wait until pin PD0 goes high (start of echo pulse)
	unsigned long ticks = get_ticks();  // get the current system time in ticks
	while (is_digital_input_high(IO_D0));  // wait until pin PD0 goes low (end of echo pulse)
	ticks = get_ticks() - ticks;  // length of the echo pulse in ticks (units are 0.4 us)
	return ticks;
}	

void calib() { // check starting distance, calibrate edge detectors
	start_time = get_time_from_sensor(); // not used in final version

	read_line_sensors(edge_sensors, IR_EMITTERS_ON); // not used in final version
	edge_calib[0] = edge_sensors[0] + edgethresh;
	edge_calib[1] = edge_sensors[1] + edgethresh;
	edge_calib[2] = edge_sensors[2] + edgethresh;
	edge_calib[3] = edge_sensors[3] + edgethresh;
	edge_calib[4] = edge_sensors[4] + edgethresh;
}

int main() {
	setup(); // play sound, wait for button B
	calib(); // calibrate distance sensor and edge sensors
	int j = 1; // keeps track of how long robot has been moving normally
	int dir = 3; // 0=stop, 1=hardleft, 2=left, 3=straight, 4=right, 5=hardright
	while(1) {
		if(j % 300 == 0) { // pick a new direction
			dir = (rand() % 5) + 1; // don't choose 0
		}
		
		time = get_time_from_sensor();
		if(time < min_dist) {// double-check
			time = get_time_from_sensor();
		}

		if(time < min_dist) { // follower too close //(time < start_time - distthresh)
			play_from_program_space(followed);
			set_digital_output(IO_C5, LOW); // turn LEDs off
			
			// spin back and forth
			set_motors(0, 0);
			delay_ms(500);
			set_motors(100, -100);
			delay_ms(300);
			set_motors(0, 0);
			delay_ms(100);
			set_motors(-100, 100);
			delay_ms(600);
			set_motors(0, 0);
			delay_ms(100);
			set_motors(100, -100);
			delay_ms(300);
			set_motors(0, 0);
			delay_ms(100);
			
			// shiver forward and backward
			delay_ms(100);
			for(i = 0; i < 15; i++) {
				set_motors(100, 100);
				delay_ms(20);
				set_motors(-100, -100);
				delay_ms(20);
			}
			set_motors(0, 0);
			
			set_digital_output(IO_C5, HIGH); // turn LEDs back on
			play_from_program_space(ledon);
			delay_ms(1000); // wait a second for follower to catch up
			continue;
		}
		stop_playing();


/*
		// back up and turn if edge detected (not used in final version)
		read_line_sensors(edge_sensors, IR_EMITTERS_ON);
		if(edge_sensors[0] > edge_calib[0] ||
		   edge_sensors[1] > edge_calib[1] ||
		   edge_sensors[2] > edge_calib[2] ||
		   edge_sensors[3] > edge_calib[3] ||
		   edge_sensors[4] > edge_calib[4]) { // on edge
			play_from_program_space(backup);
			set_motors(-max_speed, -max_speed);
			delay_ms(4*max_speed);
			set_motors(-max_speed, max_speed);
			delay_ms(4*max_speed);
			stop_playing();
			continue;
		}

*/
		set_digital_output(IO_C5, HIGH); // make sure LEDs stay on in normal operation

		if(dir == 0) { set_motors(0, 0); }
		if(dir == 1) { set_motors(max_speed/2, max_speed); }
		if(dir == 2) { set_motors(3*max_speed/4, max_speed); }
		if(dir == 3) { set_motors(max_speed, max_speed); }
		if(dir == 4) { set_motors(max_speed, 3*max_speed/4); }
		if(dir == 5) { set_motors(max_speed, max_speed/2); }

		j = (j+1);
	}
}
