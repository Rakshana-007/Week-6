# Week-6

package com.rakshanamart.servlet;

import com.rakshanamart.util.DBConnection;

import javax.servlet.annotation.WebServlet;
import javax.servlet.http.*;
import java.io.IOException;
import java.math.BigDecimal;
import java.sql.*;

@WebServlet("/payment")
public class PaymentServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest request,
                          HttpServletResponse response)
            throws IOException {

        response.setContentType("text/plain;charset=UTF-8");

        HttpSession session = request.getSession(false);
        if (session == null || session.getAttribute("userEmail") == null) {
            response.sendError(401, "Please log in first.");
            return;
        }

        String email = (String) session.getAttribute("userEmail");
        String orderValue = request.getParameter("orderId");
        String method = request.getParameter("paymentMethod");

        if (orderValue == null || method == null ||
            !method.matches("(?i)UPI|CARD|COD")) {
            response.sendError(400, "Invalid order or payment method.");
            return;
        }

        int orderId;
        try {
            orderId = Integer.parseInt(orderValue);
            if (orderId <= 0) throw new NumberFormatException();
        } catch (NumberFormatException e) {
            response.sendError(400, "Invalid order ID.");
            return;
        }

        String sql =
            "SELECT o.total_amount FROM orders o " +
            "JOIN customers c ON o.customer_id = c.id " +
            "WHERE o.id = ? AND c.email = ? AND o.order_status = 'PENDING'";

        try (Connection con = DBConnection.getConnection()) {
            con.setAutoCommit(false);

            BigDecimal amount;

            try (PreparedStatement ps = con.prepareStatement(
                    sql + " FOR UPDATE")) {
                ps.setInt(1, orderId);
                ps.setString(2, email);

                try (ResultSet rs = ps.executeQuery()) {
                    if (!rs.next()) {
                        con.rollback();
                        response.sendError(404,
                                "Pending order not found for this account.");
                        return;
                    }
                    amount = rs.getBigDecimal("total_amount");
                }
            }

            // Prevent multiple payment records for the same order.
            try (PreparedStatement ps = con.prepareStatement(
                    "SELECT id FROM payments WHERE order_id = ? FOR UPDATE")) {
                ps.setInt(1, orderId);

                try (ResultSet rs = ps.executeQuery()) {
                    if (rs.next()) {
                        con.rollback();
                        response.sendError(409,
                                "A payment record already exists for this order.");
                        return;
                    }
                }
            }

            try (PreparedStatement ps = con.prepareStatement(
                    "INSERT INTO payments " +
                    "(order_id, amount, payment_method, payment_status) " +
                    "VALUES (?, ?, ?, 'PENDING')")) {
                ps.setInt(1, orderId);
                ps.setBigDecimal(2, amount);
                ps.setString(3, method.toUpperCase());
                ps.executeUpdate();
            }

            con.commit();
            response.setStatus(HttpServletResponse.SC_CREATED);
            response.getWriter().printf(
                    "Payment record created as PENDING. Order ID: %d, Amount: %s. No money was charged.",
                    orderId, amount.toPlainString());

        } catch (SQLException e) {
            getServletContext().log("Payment record creation failed", e);
            response.sendError(500, "Unable to save payment record.");
        }
    }
}
