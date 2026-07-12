"""
DifferentialDrive.py
---------------------
High-level differential-drive controller for a 4-wheel BO-motor robot
built on an ESP32 + TB6612FNG.

Hardware Context
-----------------
Four BO motors are grouped into two independently controlled sides:
    - LEFT  side: front-left + rear-left motors, wired in parallel to
      TB6612FNG channel A.
    - RIGHT side: front-right + rear-right motors, wired in parallel to
      TB6612FNG channel B.

This is the standard "skid-steer" / differential-drive configuration:
steering is achieved purely by varying the relative speed of the left
and right sides -- there is no separate steering actuator.

The TB6612FNG has a single STBY pin shared by both channels. It is
therefore owned and managed here, at the DifferentialDrive level, NOT
by the individual MotorDriver objects.

Design Decisions
-----------------
1. Composition over inheritance: DifferentialDrive HOLDS two
   MotorDriver instances rather than subclassing MotorDriver. This
   keeps "one motor channel" (MotorDriver) and "two-wheel robot
   kinematics" (DifferentialDrive) as cleanly separated concerns, which
   makes both easier to unit-test and easier to extend later (e.g.
   swapping in a mecanum-drive class without touching MotorDriver).

2. Non-blocking: every method here is a direct, synchronous hardware
   write with no delays. move_forward(), turn_left(), etc. all return
   immediately after issuing the command. Timed behaviors ("drive
   forward for 2 seconds") belong in the caller's own scheduling logic
   (e.g. uasyncio tasks or a state machine), not inside this class.

3. PID-ready: set_speed(left, right) is the single low-level entry
   point a PID / velocity-mixing layer needs. The named convenience
   methods (move_forward, turn_left, etc.) are built ON TOP OF
   set_speed() at a fixed default duty, intended for simple open-loop
   or manual/teleop control. A closed-loop heading/velocity controller
   can bypass them entirely and call set_speed() directly every tick.

4. Symmetric point-turns: turn_left() / turn_right() spin the two
   sides in opposite directions at equal magnitude, giving the
   tightest possible turning radius for a skid-steer chassis. Gentler
   "arc" turns (turning while still moving forward) can be achieved by
   the caller via set_speed() with unequal, same-sign values instead --
   deliberately left out of the convenience API to keep it simple.

5. Central STBY ownership: standby() here toggles the ONE physical
   STBY pin shared by the entire TB6612FNG chip. This is the
   electrically correct place to manage it, and avoids the two
   MotorDriver objects either fighting over, or redundantly
   duplicating control of, the same physical pin.
"""

from machine import Pin
from Motor.MotorDriver import MotorDriver


class DifferentialDrive:
    """
    Coordinates two MotorDriver instances (left/right) to implement
    differential-drive robot motion for a 4-wheel BO-motor chassis.
    """

    # Default duty-cycle percentage used by the named convenience
    # methods (move_forward, turn_left, etc.) when the caller doesn't
    # specify a custom speed. Kept moderate (rather than 100) to reduce
    # startup current spikes and wheel slip on small BO motors.
    DEFAULT_SPEED = 60.0

    def __init__(self, left_in1, left_in2, left_pwm,
                 right_in1, right_in2, right_pwm,
                 standby_pin, default_speed=None):
        """
        Args:
            left_in1, left_in2, left_pwm (int):
                GPIO pins for the LEFT TB6612FNG channel, driving the
                front-left + rear-left motors (wired in parallel).
            right_in1, right_in2, right_pwm (int):
                GPIO pins for the RIGHT TB6612FNG channel, driving the
                front-right + rear-right motors (wired in parallel).
            standby_pin (int):
                GPIO pin connected to the TB6612FNG's single shared
                STBY line.
            default_speed (float, optional):
                Overrides DEFAULT_SPEED for this instance's convenience
                methods (move_forward, turn_left, etc.).

        Design note:
            The two MotorDriver instances are created WITHOUT their own
            standby_pin (it defaults to None inside MotorDriver) because
            STBY is shared hardware, centrally managed right here via
            self._standby -- avoiding duplicate/conflicting control of
            one physical pin from two objects.
        """
        self.left = MotorDriver(left_in1, left_in2, left_pwm, name="left")
        self.right = MotorDriver(right_in1, right_in2, right_pwm, name="right")

        self._standby = Pin(standby_pin, Pin.OUT)
        self.default_speed = (
            default_speed if default_speed is not None else self.DEFAULT_SPEED
        )

        # Enable the driver by default so the robot responds immediately
        # to motion commands. Call standby(False) explicitly if a
        # low-power idle state is desired instead.
        self._standby.value(1)

    # ------------------------------------------------------------------
    # Low-level, PID-facing interface
    # ------------------------------------------------------------------

    def set_speed(self, left_speed, right_speed):
        """
        Directly set independent speeds for the left and right sides.

        Args:
            left_speed (float): [-100.0, 100.0], sign = direction.
            right_speed (float): [-100.0, 100.0], sign = direction.

        Design note:
            This is the ONLY method a PID / velocity controller needs.
            Calling it every control tick with two independent signed
            values is sufficient to implement arbitrary linear +
            angular velocity mixing, e.g. the standard differential-
            drive mixing equations computed by the caller:

                left_speed  = linear_velocity - angular_velocity
                right_speed = linear_velocity + angular_velocity

            Keeping mixing logic OUT of this class means DifferentialDrive
            doesn't need to know anything about units (m/s, rad/s, etc.)
            or about whatever PID library is used upstream.
        """
        self.left.set_speed(left_speed)
        self.right.set_speed(right_speed)

    # ------------------------------------------------------------------
    # High-level convenience / teleop interface
    # ------------------------------------------------------------------

    def move_forward(self, speed=None):
        """
        Drive both sides forward at equal speed.

        Args:
            speed (float, optional): Duty percentage [0-100]. Defaults
                to self.default_speed. Sign is ignored -- direction is
                determined by the method name itself.
        """
        s = self._resolve_speed(speed)
        self.set_speed(s, s)

    def move_backward(self, speed=None):
        """
        Drive both sides in reverse at equal speed.

        Args:
            speed (float, optional): Duty percentage [0-100]. Defaults
                to self.default_speed.
        """
        s = self._resolve_speed(speed)
        self.set_speed(-s, -s)

    def turn_left(self, speed=None):
        """
        Rotate in place to the left (counter-clockwise viewed from
        above): left side reverses while the right side drives forward.

        Design note:
            A point-turn (equal magnitude, opposite sign on each side)
            gives the smallest possible turning radius and the most
            predictable behavior for open-loop/manual control -- the
            robot rotates essentially about its own center. Gentler
            "arc" turns are intentionally NOT provided here; they are a
            natural job for the low-level set_speed() call with
            unequal, same-sign values, typically driven by a PID
            heading controller rather than a fixed teleop command.
        """
        s = self._resolve_speed(speed)
        self.set_speed(-s, s)

    def turn_right(self, speed=None):
        """
        Rotate in place to the right (clockwise viewed from above):
        left side drives forward while the right side reverses.
        """
        s = self._resolve_speed(speed)
        self.set_speed(s, -s)

    def stop_robot(self):
        """
        Stop both sides by coasting (free-spin, no active braking).

        Design note:
            Named and implemented distinctly from brake() because
            coasting and braking are different physical behaviors.
            stop_robot() is the "gentle stop" used for ordinary
            end-of-motion; brake() is reserved for situations needing a
            fast, firm stop (e.g. an obstacle detected by a sensor).
        """
        self.left.coast()
        self.right.coast()

    def brake(self):
        """
        Actively brake both motor channels (short-brake), bringing the
        robot to a stop as quickly as the hardware allows.

        Use this for emergency stops, or any position-critical stop
        where coasting would let the robot overshoot its target.
        """
        self.left.brake()
        self.right.brake()

    def standby(self, enable):
        """
        Enable or disable the entire TB6612FNG chip via its single
        shared STBY pin.

        Args:
            enable (bool): True  -> chip active.
                            False -> chip in low-power standby; ALL
                                     outputs (both channels) go
                                     high-impedance and motors free-spin
                                     regardless of any IN1/IN2/PWM state.

        Design note:
            This is the correct, single point of control for STBY,
            since it is one physical pin shared by both channels on the
            TB6612FNG. Disabling it is a STRONGER form of "stop" than
            either stop_robot() or brake() -- it also cuts the driver
            IC's own power draw, which is useful for battery-saving
            idle periods (e.g. the robot has been idle/unreachable for
            N seconds).
        """
        self._standby.value(1 if enable else 0)
        if not enable:
            # Reflect the disabled state in cached speed so any code
            # reading get_speed() afterwards sees zero rather than a
            # stale pre-standby value.
            self.left._current_speed = 0.0
            self.right._current_speed = 0.0

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    def _resolve_speed(self, speed):
        """
        Internal helper: resolve an optional speed argument against the
        instance default, and take its absolute value, since direction
        for the convenience methods is determined entirely by the
        method name (move_forward vs. move_backward, etc.) rather than
        by the sign of the speed argument.
        """
        if speed is None:
            speed = self.default_speed
        return abs(speed)