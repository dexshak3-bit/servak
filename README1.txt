                if (stock != null) stock.Quantity++;
                item.Quantity--;
                App.Db.SaveChanges();
                RefreshItems();
            }
            else
            {
                await RemoveItem(item);
            }
        }
    }

    private async void BtnRemove_Click(object? sender, RoutedEventArgs e)
    {
        if (sender is Button btn && btn.Tag is OrderItem item)
        {
            await RemoveItem(item);
        }
    }

    private async Task RemoveItem(OrderItem item)
    {
        bool ok = await Dialogs.Confirm(this, $"Удалить позицию «{item.ProductName}» из заказа?");
        if (!ok) return;

        var stock = App.Db.StockItems.Find(item.StockItemId);
        if (stock != null)
        {
            stock.Quantity += item.Quantity;
        }

        App.Db.OrderItems.Remove(item);
        _order.OrderItems.Remove(item);
        App.Db.SaveChanges();
        RefreshItems();
    }

    private void BtnClose_Click(object? sender, RoutedEventArgs e) => Close();
}
