// follows light from another 3pi
// spins back and forth if lost
// 3 photoresistors (1 unused), edge sensors

#include </home/craig/Downloads/libpololu-avr/pololu/3pi.h>
#include <pololu/3pi.h>

// 3pi buzzer sounds
const char welcome[] PROGMEM = ">g32>>c32";
const char go[] PROGMEM = "L16 cdegreg4";
const char lost[] PROGMEM = "c8<b8";
const char found[] PROGMEM = ">c32>d32>c32b32>c32";
// declare variables
int start_ls, start_rs, start_bs, follow_d, follow_ls, follow_rs, follow_bs, ls, rs, bs;
int l_turn, f_move;
int left_wheel, right_wheel;
int max_speed = 150;
unsigned int edge_sensors[5];
int edgethresh = 500;
int edge_calib[5];
int islost = 0;

int i;

void wait_for_buttonB() {
	// wait for button press, play welcome sound
	while(!button_is_pressed(BUTTON_B)) { // wait for button B to be pressed 
		delay_ms(10);
	}
	play_from_program_space(welcome);
	while(button_is_pressed(BUTTON_B)) { // wait for button B to be released
		delay_ms(10);
	}
}

void setup() {
	// play music
	pololu_3pi_init(2000); // needed for using sensors, sets timeout value
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

void darkcalib() { // read in light levels of the dark room background
	start_ls = analog_read_millivolts(6);
	start_rs = analog_read_millivolts(7);
	start_bs = analog_read_millivolts(PC5); // not used in final version

	read_line_sensors(edge_sensors, IR_EMITTERS_ON);
	edge_calib[0] = edge_sensors[0] + edgethresh;
	edge_calib[1] = edge_sensors[1] + edgethresh;
	edge_calib[2] = edge_sensors[2] + edgethresh;
	edge_calib[3] = edge_sensors[3] + edgethresh;
	edge_calib[4] = edge_sensors[4] + edgethresh;
}

void calib() { // read in light levels of object to follow
	wait_for_buttonB();
	
	follow_ls = analog_read_millivolts(6);
	follow_rs = analog_read_millivolts(7);
	follow_bs = analog_read_millivolts(PC5); // not used in final version
	follow_d = follow_ls + follow_rs; // not used in final version
}

int main() {
	setup(); // play sound, wait for button B
	darkcalib(); // keep track of initial dark photoresistor values
	calib(); // wait for button B, calibrate to brightness of object to follow
	while(1) {
		ls = analog_read_millivolts(6);
		rs = analog_read_millivolts(7);
		bs = analog_read_millivolts(PC5); // not used in final version
		
		if(ls < start_ls + 30 || rs < start_rs + 30) { //(ls < start_ls + 80 && rs < start_rs + 80)
			islost = islost + 1;
			//if(!is_playing()) {
			//	play_from_program_space(lost);
			//}
			if(islost < 100) { set_motors(0, 0); }
			if((islost + 50) % 200 < 100) { set_motors(50, -50); }
			else { set_motors(-50, 50); }
			delay_ms(10);
			continue;
		}
/*
		if((ls < start_ls + 40 || rs < start_rs + 40) && (islost == 0)) {
			islost = 1;
			//if(!is_playing()) {
			//	play_from_program_space(lost);
			//}
			set_motors(-max_speed/4, -max_speed/4);
			delay_ms(1000);
			set_motors(0, 0);
			
			i = 0;
			while(i < 200 && (ls < start_ls + 40 || rs < start_rs + 40)) {
				ls = analog_read_millivolts(6);
				rs = analog_read_millivolts(7);
				delay_ms(10);
			}
			islost = 1;
			continue;
		}
		if((ls < start_ls + 40 || rs < start_rs + 40) && (islost == 1)) {
			//if(!is_playing()) {
			//	play_from_program_space(lost);
			//}
			set_motors(-max_speed/2, max_speed/2);
			continue;
		}	
*/
		//if(islost == 1) {
		//	stop_playing();
		//	play_from_program_space(found);
		//}
		islost = 0; // robot is not lost
		
		l_turn = (ls - follow_ls) - (rs - follow_rs); // amount to turn
		f_move = -(ls - follow_ls) - (rs - follow_rs); // amount to move forward

		read_line_sensors(edge_sensors, IR_EMITTERS_ON); // check if on an edge

		for(i = 0; i < 5; i++) {
			if(edge_sensors[i] > edge_calib[i] && f_move > 0) { // on edge, moving forward
				f_move = 0; // don't move forward
			}
		}

		left_wheel = (f_move / 3) - (l_turn / 8);
		right_wheel = (f_move / 3) + (l_turn / 8);

		// don't move faster than max_speed
		if(left_wheel > max_speed) { left_wheel = max_speed; }
		if(left_wheel < -max_speed) { left_wheel = -max_speed; }
		if(right_wheel > max_speed) { right_wheel = max_speed; }
		if(right_wheel < -max_speed) { right_wheel = -max_speed; }

		set_motors(left_wheel, right_wheel);
	}
}
