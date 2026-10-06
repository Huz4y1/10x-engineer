  

1. frontend displays bin QR code scanner
2. user presses scan
3. image of QR code converted into digits
4. digits sent to backend
5. backend validates and checks if in database
6. if in database successful response sent back
7. Frontend displays waste barcode scanner
8. client presses scan button
9. image of barcode is converted into digits

1. digits sent to backend
2. backend validates via database
3. if not in database via open food api
4. open food api sends response and result stored in database for caching
5. if result is successful points are added to user in database
6. points from database are displayed on main menu frontend