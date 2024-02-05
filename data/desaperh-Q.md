Description :

Using some specific parameters, the multiplier from controller will never be updated.


POC:

Modifying the utils.cairo from controller with the following parameters :
// Default controller parameters
const P_GAIN: u128 = 100000000000000000000000000000; // 100 * RAY_ONE 
const I_GAIN: u128 = 0; 
const ALPHA_P: u8 = 6; 
const BETA_P: u8 = 6;
const ALPHA_I: u8 = 6; 
const BETA_I: u8 = 6; 

The test "test_frequent_updates" will be false with error message "Multiplier should increase".

Recommendation:

Add a require statement to the controller contract to restring the range of the constants ALPHA_P, BETA_P, ALPHA_I and BETA_I