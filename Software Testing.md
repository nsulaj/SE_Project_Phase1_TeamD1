Task Overview: Choose 2-3 critical classes, or functions from your software, prepare comprehensive test cases, write the related test code, execute the tests, and document the results clearly in your report.
1. Introduction to Testing
●	Define software testing as the process of evaluating software to identify defects, bugs, or unexpected behavior.
Software testing is the process of evaluating a program or system to discover defects, bugs, or unexpected behavior. It involves running the software and observing how it performs in order to check whether it matches the expected results. In simple terms, testing is about making sure the software works the way it is supposed to.
	
●	Highlight why testing is important in software development, especially for improving reliability, correctness, and maintainability.
Testing is a crucial part of software development because it improves the overall quality of the product. It helps ensure reliability, meaning the system can be trusted to perform consistently. It also checks correctness by confirming that the software meets its requirements and functions properly. In addition, testing supports maintainability, since identifying and fixing issues early makes the software easier to update and manage in the future. Without proper testing, software may fail in real-world situations and create serious problems for users.
2. Purpose of Testing
●	Explain that testing helps identify defects early in the development process.

The purpose of testing is to identify defects as early as possible in the development process. Finding problems early makes them easier and less costly to fix compared to discovering them after the software has been released. 

●	Show how testing verifies that software components behave as intended under expected and unexpected conditions.

Testing also ensures that software behaves as intended. It verifies that each component works correctly under normal conditions and checks how the system reacts under unexpected situations, such as incorrect inputs or high usage. By doing this, testing not only confirms that the system meets its requirements but also improves its overall quality and reliability. In general, testing acts as a safeguard that helps developers deliver software that is stable, functional, and ready for real users. 
3. Focus on Testing a Single Component - Booking Service
The selected component for testing is the BookingService integration functionality. This component is important because it controls the main booking process of the sports facility system. It connects the citizen, facility, schedule, booking, and payment parts of the application. If this part does not work correctly, citizens may not be able to book a facility, or the system may allow incorrect bookings.
The BookingService was selected because it has a direct effect on the main purpose of the system. It checks whether the facility exists, whether the selected date and time slot are valid, whether the slot is already booked, and whether the booking can be cancelled. It also creates the payment record and updates the available slots after a booking is cancelled.
This test is an integration test because it runs with the Spring Boot application context and uses real repositories for saving and reading test data. This makes the test closer to the real system behavior compared to a simple unit test.

4. Preparing Test Cases
The test cases were designed to cover the most important booking situations. They include successful booking creation, duplicate booking prevention, available slot checking, cancellation, and invalid input cases. The goal was to make sure that the BookingService works correctly under normal and incorrect conditions.
| Test ID | Scenario | Input | Expected Result |
|---|---|---|---|
| TC01 | Valid booking creation | Valid facility, future date, available time slot | Booking is created successfully with pending status and cash payment |
| TC02 | Duplicate booking | Two users try to book the same facility, date, and time slot | Second booking is rejected with “This slot is already booked.” |
| TC03 | Available slot check | One slot is already booked | Booked slot is removed from the available slots list |
| TC04 | Booking cancellation | Citizen cancels an existing booking | Booking status becomes CANCELLED and the slot becomes available again |
| TC05 | Invalid facility ID | Facility ID does not exist | Booking is rejected with “Facility not found” |
| TC06 | Past booking date | Booking date is in the past | Booking is rejected because the date must be today or future |
| TC07 | Null time slot | Time slot is not selected | Booking is rejected because the selected slot is not available |
| TC08 | Empty time slot | Time slot is empty | Booking is rejected because the selected slot is not available |
5. Writing Test Code

@SpringBootTest
class BookingServiceIntegrationTest {

    @Autowired
    private BookingService bookingService;

    @Autowired
    private BookingRepository bookingRepository;

    @Autowired
    private FacilityRepository facilityRepository;

    @Autowired
    private FacilityScheduleRepository scheduleRepository;

    @Autowired
    private PaymentRepository paymentRepository;

    @Autowired
    private UserRepository userRepository;

    @MockBean
    private BookingNotificationService bookingNotificationService;

    @MockBean
    private JavaMailSender javaMailSender;

    private User citizen;
    private User secondCitizen;
    private Facility facility;
    private LocalDate bookingDate;

    @BeforeEach
    void setUp() {
        paymentRepository.deleteAll();
        bookingRepository.deleteAll();
        scheduleRepository.deleteAll();
        facilityRepository.deleteAll();
        userRepository.deleteAll();

        reset(bookingNotificationService);

        citizen = userRepository.save(user("citizen@example.test"));
        secondCitizen = userRepository.save(user("other-citizen@example.test"));
        facility = facilityRepository.save(facility());

        bookingDate = LocalDate.now().plusDays(7);
        scheduleRepository.save(schedule(facility, bookingDate));
    }

    @Test
    void createBookingWithValidFacilityDateAndTimeSlotSucceeds() {
        BookingForm form = bookingForm(facility.getId(), bookingDate, "08:00-09:00");

        Booking booking = bookingService.createBooking(
            citizen,
            facility.getId(),
            form,
            bindingResult(form)
        );

        assertAll(
            () -> assertNotNull(booking.getId()),
            () -> assertEquals(citizen.getId(), booking.getUser().getId()),
            () -> assertEquals(facility.getId(), booking.getFacility().getId()),
            () -> assertEquals(bookingDate, booking.getBookingDate()),
            () -> assertEquals("08:00-09:00", booking.getTimeSlot()),
            () -> assertEquals(BookingStatus.PENDING, booking.getStatus()),
            () -> assertEquals(
                BookingVerificationStatus.UNVERIFIED,
                booking.getVerificationStatus()
            ),
            () -> assertEquals(
                PaymentMethod.CASH_AT_FACILITY,
                booking.getPayment().getMethod()
            ),
            () -> assertEquals(
                PaymentStatus.CASH_PENDING,
                booking.getPayment().getStatus()
            )
        );

        verify(bookingNotificationService).sendBookingCreated(booking);
    }

    @Test
    void createBookingForSameFacilityDateAndTimeSlotFails() {
        BookingForm firstForm = bookingForm(
            facility.getId(),
            bookingDate,
            "08:00-09:00"
        );

        bookingService.createBooking(
            citizen,
            facility.getId(),
            firstForm,
            bindingResult(firstForm)
        );

        BookingForm duplicateForm = bookingForm(
            facility.getId(),
            bookingDate,
            "08:00-09:00"
        );

        IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> bookingService.createBooking(
                secondCitizen,
                facility.getId(),
                duplicateForm,
                bindingResult(duplicateForm)
            )
        );

        assertEquals("This slot is already booked.", exception.getMessage());
        assertEquals(1, bookingRepository.count());
    }

    @Test
    void getAvailableSlotsExcludesAlreadyBookedSlot() {
        BookingForm form = bookingForm(
            facility.getId(),
            bookingDate,
            "08:00-09:00"
        );

        bookingService.createBooking(
            citizen,
            facility.getId(),
            form,
            bindingResult(form)
        );

        List<String> availableSlots = bookingService.getAvailableSlots(
            facility.getId(),
            bookingDate
        );

        assertAll(
            () -> assertFalse(availableSlots.contains("08:00-09:00")),
            () -> assertTrue(availableSlots.contains("09:00-10:00"))
        );
    }

    @Test
    void cancelBookingUpdatesStatusAndReleasesSlot() {
        BookingForm form = bookingForm(
            facility.getId(),
            bookingDate,
            "08:00-09:00"
        );

        Booking booking = bookingService.createBooking(
            citizen,
            facility.getId(),
            form,
            bindingResult(form)
        );

        bookingService.cancelBooking(citizen, booking.getId());

        Booking cancelled = bookingRepository
            .findById(booking.getId())
            .orElseThrow();

        assertAll(
            () -> assertEquals(BookingStatus.CANCELLED, cancelled.getStatus()),
            () -> assertEquals(null, cancelled.getActiveSlotKey()),
            () -> assertTrue(
                bookingService
                    .getAvailableSlots(facility.getId(), bookingDate)
                    .contains("08:00-09:00")
            )
        );
    }

    @Test
    void createBookingWithInvalidFacilityIdFails() {
        BookingForm form = bookingForm(
            999_999L,
            bookingDate,
            "08:00-09:00"
        );

        IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> bookingService.createBooking(
                citizen,
                999_999L,
                form,
                bindingResult(form)
            )
        );

        assertEquals("Facility not found", exception.getMessage());
        assertEquals(0, bookingRepository.count());
    }

    @Test
    void createBookingWithPastDateFails() {
        LocalDate pastDate = LocalDate.now().minusDays(1);

        BookingForm form = bookingForm(
            facility.getId(),
            pastDate,
            "08:00-09:00"
        );

        IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> bookingService.createBooking(
                citizen,
                facility.getId(),
                form,
                bindingResult(form)
            )
        );

        assertEquals(
            "Booking date must be today or a future date.",
            exception.getMessage()
        );

        assertEquals(0, bookingRepository.count());
    }

    @Test
    void createBookingWithNullTimeSlotFails() {
        BookingForm form = bookingForm(
            facility.getId(),
            bookingDate,
            null
        );

        IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> bookingService.createBooking(
                citizen,
                facility.getId(),
                form,
                bindingResult(form)
            )
        );

        assertEquals(
            "Selected time slot is not available for this facility.",
            exception.getMessage()
        );

        assertEquals(0, bookingRepository.count());
    }

    @Test
    void createBookingWithEmptyTimeSlotFails() {
        BookingForm form = bookingForm(
            facility.getId(),
            bookingDate,
            ""
        );

        IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> bookingService.createBooking(
                citizen,
                facility.getId(),
                form,
                bindingResult(form)
            )
        );

        assertEquals(
            "Selected time slot is not available for this facility.",
            exception.getMessage()
        );

        assertEquals(0, bookingRepository.count());
    }

    private BookingForm bookingForm(Long facilityId, LocalDate date, String timeSlot) {
        BookingForm form = new BookingForm();
        form.setFacilityId(facilityId);
        form.setBookingDate(date);
        form.setTimeSlot(timeSlot);
        form.setPaymentMethod(PaymentMethod.CASH_AT_FACILITY);
        return form;
    }

    private BeanPropertyBindingResult bindingResult(BookingForm form) {
        return new BeanPropertyBindingResult(form, "bookingForm");
    }

    private User user(String email) {
        User user = new User();
        user.setFullName("Test Citizen");
        user.setEmail(email);
        user.setPassword("{noop}Password123!");
        user.setRole(Role.CITIZEN);
        user.setActive(true);
        user.setEmailVerified(true);
        return user;
    }

    private Facility facility() {
        Facility facility = new Facility();
        facility.setName("Integration Test Court");
        facility.setType("Tennis");
        facility.setLocation("Test Location");
        facility.setDescription("Facility used by booking service integration tests.");
        facility.setPrice(BigDecimal.valueOf(25));
        facility.setActive(true);
        return facility;
    }

    private FacilitySchedule schedule(Facility facility, LocalDate date) {
        FacilitySchedule schedule = new FacilitySchedule();
        schedule.setFacility(facility);
        schedule.setDayOfWeek(date.getDayOfWeek());
        schedule.setOpenTime(LocalTime.of(8, 0));
        schedule.setCloseTime(LocalTime.of(10, 0));
        schedule.setSlotDurationMinutes(60);
        schedule.setActive(true);
        return schedule;
    }
}
6. Running Tests
The tests were executed from IntelliJ IDEA by running the BookingServiceIntegrationTest class. They can also be executed from the terminal using the Maven command mvn test. When the tests pass, IntelliJ shows green check marks beside the test methods and the process finishes with exit code 0.
For this test class, 8 integration test cases should be executed. A successful result means that all 8 tests passed without failures or errors. A failed result would mean that at least one expected result did not match the actual system behavior. An error would usually mean that the test could not run because of a setup, configuration, or database problem.
<img width="940" height="246" alt="image" src="https://github.com/user-attachments/assets/18bf53ed-48d5-4200-9025-647f623df2a0" />
7. Test Coverage
The integration tests cover the most important paths of the BookingService. They test the successful booking path, the prevention of duplicate bookings, the checking of available slots, booking cancellation, and several invalid input cases. This gives confidence that the booking process works correctly with the database and related entities.
The tests also cover important failure cases. These include invalid facility ID, past booking date, null time slot, and empty time slot. In each of these cases, the test confirms that the system rejects the request and does not save an incorrect booking in the database.
One limitation is that these tests mainly focus on cash-at-facility bookings. More tests could be added later for credit card authorization, staff verification, booking updates, completed bookings, and email notification content. Even with this limitation, the current tests cover the main booking process and the most important validation rules.
